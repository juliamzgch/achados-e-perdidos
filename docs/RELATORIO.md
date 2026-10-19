# Relatório técnico

Este relatório explica como a aplicação fornecida foi empacotada e como os serviços do Compose cooperam. O código da API e o esquema SQL foram mantidos conforme os arquivos fornecidos; as mudanças desta entrega concentram-se na imagem, na rede, nos volumes, nas configurações e na documentação.

## Visão geral e rastreabilidade

A execução local separa responsabilidades: PostgreSQL mantém os registros, Flask/Gunicorn implementa a API, Nginx é a entrada HTTP e Adminer é uma ferramenta administrativa opcional. Os dados mutáveis ficam em volumes; a rede definida pelo Compose permite que os serviços se encontrem por nome, sem publicar o banco ou a aplicação diretamente no host.

| Requisito ou aspecto avaliado | Implementação | Evidência principal |
| --- | --- | --- |
| Imagem Python para a aplicação | [`app/Dockerfile`](../app/Dockerfile): Python 3.12 slim, instalação de dependências em camada própria e execução como usuário `app`. | [`03-inspecoes.txt`](evidencias/03-inspecoes.txt) |
| Banco e inicialização | Serviço `db`, healthcheck `pg_isready` e script [`db/init.sql`](../db/init.sql). | [`01-subida.txt`](evidencias/01-subida.txt) |
| Persistência do banco e das fotos | Volumes nomeados `dados_banco` e `fotos`, preservados por `docker compose down`. | [`04-persistencia.txt`](evidencias/04-persistencia.txt) |
| API e cadastro com imagem | Gunicorn executa Flask; o caminho de gravação é `/app/uploads`, montado em `fotos`. | [`02-testes-funcionais.txt`](evidencias/02-testes-funcionais.txt) |
| Proxy reverso e entrada HTTP | Nginx publica `8080:80`; `web` e `db` não publicam portas no host. | [`06-extras-nginx-adminer.txt`](evidencias/06-extras-nginx-adminer.txt) |
| Upload maior que o limite padrão do Nginx | `client_max_body_size 10M`; teste com PNG maior que 1 MB, download e comparação byte a byte. | [`08-limite-upload-nginx.txt`](evidencias/08-limite-upload-nginx.txt) |
| Administração opcional | Adminer isolado no profile `debug`, publicado somente em loopback. | [`06-extras-nginx-adminer.txt`](evidencias/06-extras-nginx-adminer.txt) |
| Cópia e restauração das imagens | Backup TAR.GZ restaurado em volume separado, com igualdade dos hashes SHA-256. | [`07-backup-restauracao.txt`](evidencias/07-backup-restauracao.txt) |

O [índice de evidências](evidencias/README.md) descreve o conteúdo de cada registro. Os arquivos são saídas de testes realizados, não um monitoramento contínuo nem uma promessa de disponibilidade do ambiente.

## 1. Dockerfile da aplicação

O Dockerfile está em [`app/Dockerfile`](../app/Dockerfile). A ordem das instruções preserva o cache de dependências: o manifesto Python é copiado e instalado antes de `app.py`.

| Instrução | Função e motivo |
| --- | --- |
| `FROM python:3.12-slim` | Usa a imagem oficial do Python 3.12 na variante slim solicitada. Ela fornece o runtime sem incluir uma distribuição completa desnecessária para esta aplicação. |
| `ENV PYTHONDONTWRITEBYTECODE=1` | Evita gerar arquivos `.pyc` durante a execução e reduz arquivos temporários no contêiner. |
| `ENV PYTHONUNBUFFERED=1` | Faz com que a saída Python, incluindo os logs de Gunicorn, seja enviada sem buffering prolongado. |
| `WORKDIR /app` | Define `/app` como diretório corrente para cópias relativas e comandos executados na imagem. |
| `COPY requirements.txt .` | Copia primeiro somente as dependências. Se o código mudar sem alterar `requirements.txt`, esta camada e a instalação seguinte continuam elegíveis para cache. |
| `RUN pip install --no-cache-dir -r requirements.txt ...` | Instala Flask, psycopg e Gunicorn sem guardar o cache de downloads do pip na imagem. Na mesma camada, cria o grupo e usuário de sistema `app`, cria `/app/uploads` e transfere a propriedade desse diretório ao usuário. Encadear as ações com `&&` evita camadas intermediárias para cada comando. |
| `COPY --chown=app:app app.py .` | Copia o código da API com proprietário não-root. Como esta instrução vem depois da instalação, alterações no código não forçam a reinstalação das dependências. |
| `USER app` | Executa Gunicorn e a aplicação com o usuário sem privilégios, em vez de root. O diretório de uploads já está gravável por esse usuário; o uso desse usuário é conferido em [`03-inspecoes.txt`](evidencias/03-inspecoes.txt). |
| `EXPOSE 5000` | Documenta a porta interna em que Gunicorn recebe tráfego. `EXPOSE` não publica a porta no host. |
| `CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]` | Inicia o servidor Gunicorn no endereço acessível aos outros serviços Docker. A forma JSON (exec-form) evita um shell intermediário; `app:app` identifica o módulo `app.py` e o objeto Flask `app`. |

O arquivo [`app/.dockerignore`](../app/.dockerignore) exclui `.env`, `.git`, caches Python e arquivos `.pyc` do contexto de build. Como o Compose define `build.context: ./app`, o `.env` da raiz já fica fora do contexto enviado para construir essa imagem; a regra também protege contra um arquivo `.env` acidentalmente colocado dentro de `app/`. As credenciais são passadas em tempo de execução pelo Compose, não copiadas para a imagem.

### Cache de camadas na prática

Na primeira compilação, Docker executa a instalação de dependências. Se somente `app.py` mudar, as camadas de `FROM`, `WORKDIR`, `COPY requirements.txt` e `pip install` continuam iguais e podem ser reutilizadas; apenas a cópia do código e as camadas subsequentes precisam ser refeitas. Se `requirements.txt` mudar, a camada de instalação é invalidada, que é o comportamento esperado para instalar a nova dependência.

## 2. Docker Compose, serviço por serviço

O [`docker-compose.yml`](../docker-compose.yml) define os serviços `db`, `web`, `nginx` e o serviço opcional `adminer`, além de uma rede e dois volumes nomeados. Os campos de um serviço descrevem elementos diferentes: `image`/`build` selecionam como obter a imagem; `environment` injeta configuração no processo; `volumes` conecta armazenamento; `depends_on` ordena a inicialização; `healthcheck` mede prontidão; `ports` publica portas do host; `networks` define comunicação entre contêineres; `restart` define a política de reinício.

### Banco `db`

- A imagem `postgres:16-alpine` é a imagem oficial Postgres 16 na variante Alpine.
- `POSTGRES_DB`, `POSTGRES_USER` e `POSTGRES_PASSWORD` são interpoladas a partir do `.env`. A sintaxe `${VAR:?...}` faz o Compose interromper a configuração se uma variável obrigatória não estiver definida, em vez de subir o banco silenciosamente com valor vazio.
- `dados_banco:/var/lib/postgresql/data` guarda o diretório de dados fora do ciclo de vida do contêiner.
- `./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro` é um bind mount somente leitura. O prefixo `./` indica um caminho relativo ao arquivo Compose; `:ro` impede escrita pelo contêiner. A imagem oficial executa o script quando inicializa um diretório de dados vazio; a alteração posterior do arquivo não equivale a uma migração de banco.
- O healthcheck usa `pg_isready` com as credenciais e o banco configurados. Os `$$` escapam o sinal `$` para que as variáveis sejam expandidas dentro do contêiner, não durante a interpolação do Compose. `interval`, `timeout`, `retries` e `start_period` controlam a frequência, o tempo de resposta permitido, as tentativas e o período de tolerância inicial.
- Não há bloco `ports` para `db`: a porta `5432` não é publicada no host.

### Aplicação `web`

- `build.context: ./app` limita o contexto ao código da aplicação e usa o Dockerfile localizado em `app/`.
- As variáveis `DB_HOST=db`, `DB_PORT=5432`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` e `UPLOAD_DIR=/app/uploads` correspondem à configuração lida pelo `app.py`.
- `fotos:/app/uploads` guarda os arquivos recebidos em um volume nomeado.
- `depends_on` com `condition: service_healthy` espera o PostgreSQL aceitar conexões, e não apenas o início do contêiner.
- O healthcheck executa uma chamada HTTP local a `/health`. Essa rota executa uma consulta SQL; portanto, a verificação testa tanto o processo da API quanto sua capacidade de conectar ao banco. Os parâmetros do healthcheck controlam quando a aplicação é considerada saudável ou não saudável.
- O serviço não publica porta no host. A porta 5000 é acessível aos serviços da rede Docker, em especial ao Nginx. A evidência de estado e portas publicados está em [`03-inspecoes.txt`](evidencias/03-inspecoes.txt).

### Proxy reverso `nginx`

- Usa a imagem oficial Nginx fixada por digest, para que o mesmo manifesto de imagem seja usado entre builds enquanto o digest não mudar.
- Monta `./nginx/default.conf` como somente leitura em `/etc/nginx/conf.d/default.conf`.
- `8080:80` publica a interface HTTP de entrada no host. É o único serviço principal com uma porta publicada; o `web` não publica `5000` e o banco não publica `5432`.
- `depends_on` com `condition: service_healthy` espera a aplicação passar no healthcheck antes de iniciar o proxy.
- A configuração encaminha pedidos a `web:5000`, transmite cabeçalhos de origem e permite corpo de até 10 MB. O limite do Nginx não substitui validação de conteúdo nem cria autenticação.
- A evidência [`08-limite-upload-nginx.txt`](evidencias/08-limite-upload-nginx.txt) registra um cadastro real por esse caminho e a leitura posterior do arquivo.

### Painel `adminer`

- Usa a imagem oficial Adminer fixada por digest.
- `profiles: ["debug"]` exclui o painel do `docker compose up` padrão. Para iniciá-lo explicitamente, use `docker compose --profile debug up -d`.
- `ADMINER_DEFAULT_SERVER=db` sugere o hostname interno do banco na tela de login.
- A porta é publicada como `127.0.0.1:8081:8080`, acessível apenas pelo loopback do host. O Adminer participa da rede para se conectar ao banco, mas o PostgreSQL continua sem porta publicada. O HTTP 200 da interface e o estado do serviço estão registrados em [`06-extras-nginx-adminer.txt`](evidencias/06-extras-nginx-adminer.txt).

### Rede, volumes e reinício

`rede_interna` é uma rede bridge definida pelo usuário. O DNS do Docker resolve o hostname `db` para o endereço atual do serviço na rede. Os serviços principais não dependem de endereços IP codificados no código. A rede é privada no sentido de rede de contêineres do projeto, mas não está marcada como `internal: true`; isso permite publicar a porta do proxy e significa que a configuração não pretende bloquear toda saída de rede dos contêineres.

Os volumes `dados_banco` e `fotos` são gerenciados pelo Docker, não pertencem à rede e permanecem após `docker compose down`. O `restart: unless-stopped` nos serviços faz o Docker tentar reiniciá-los após falhas ou reinicialização do daemon, salvo se tiverem sido parados explicitamente.

## 3. Inicialização e dependências

Na inicialização normal, a ordem efetiva esperada é:

1. Compose cria a rede e o volume `dados_banco` (e `fotos`).
2. O PostgreSQL inicializa o diretório de dados. Se estiver vazio, cria banco/usuário e executa `init.sql`.
3. O healthcheck com `pg_isready` passa quando o banco aceita conexões.
4. O Compose inicia `web`; seu healthcheck chama `/health`, que testa a conexão com o banco.
5. Quando `web` está saudável, o Nginx inicia e publica `localhost:8080`.

Assim, `depends_on` sozinho não é confundido com prontidão: a condição `service_healthy` usa o healthcheck do serviço dependente. A saída de inicialização em [`01-subida.txt`](evidencias/01-subida.txt) registra a inicialização dos serviços, enquanto [`02-testes-funcionais.txt`](evidencias/02-testes-funcionais.txt) registra chamadas às rotas da API. O Adminer não participa desta ordem normal; é iniciado somente quando o profile `debug` é solicitado.

## 4. Caminho de um cadastro com foto

Um `POST /itens` com multipart/form-data segue este percurso:

1. O usuário envia `descricao`, `local` e `foto` para `http://localhost:8080/itens`.
2. A publicação `8080:80` entrega o pedido ao Nginx. O limite de corpo é aplicado aqui antes do encaminhamento.
3. Nginx encaminha o método, caminho, corpo e cabeçalhos para `web:5000` pela `rede_interna`.
4. Gunicorn recebe a chamada e despacha para a rota Flask `cadastrar`.
5. Flask valida a presença de `descricao` e `local`; se houver arquivo, cria um nome UUID e grava em `/app/uploads`.
6. `/app/uploads` é o ponto de montagem de `fotos`, portanto a imagem reside no volume nomeado e não na camada efêmera do contêiner `web`.
7. A aplicação conecta-se a `db:5432` usando as variáveis de ambiente e insere na tabela `itens` o texto e o nome da foto. Os dados do PostgreSQL ficam em `dados_banco`.
8. A resposta HTTP 201 contém o identificador e o nome da imagem. `GET /itens` devolve o caminho `/fotos/<nome>`; o Nginx encaminha o pedido ao endpoint Flask, que envia o arquivo do volume.

Os arquivos e as linhas do banco vivem em armazenamentos distintos e não há transação distribuída entre eles. Se o processo falhar depois de gravar o arquivo, mas antes de inserir a linha SQL, pode restar um arquivo sem registro; se a gravação falhar, a aplicação não deveria anunciar um cadastro completo. Essa separação é suficiente para demonstrar persistência na atividade, mas um sistema de produção precisaria de tratamento compensatório, limpeza periódica e monitoração. O teste ponta a ponta do cadastro e da recuperação da foto está em [`02-testes-funcionais.txt`](evidencias/02-testes-funcionais.txt).

## 5. Ciclo de vida dos volumes

| Operação | Contêineres / rede | `dados_banco` | `fotos` |
| --- | --- | --- | --- |
| `docker compose stop` | Para os serviços; mantém contêineres e rede | Mantido | Mantido |
| `docker compose start` | Inicia novamente os mesmos contêineres | Reutilizado | Reutilizado |
| `docker compose down` | Remove contêineres e rede | Mantido | Mantido |
| `docker compose up -d` após `down` | Cria contêineres e rede novos | Reutilizado; preserva itens | Reutilizado; preserva arquivos |
| `docker compose down -v` | Remove contêineres, rede e volumes do projeto | Apagado | Apagado |
| `docker compose up -d` após `down -v` | Inicialização limpa | PostgreSQL executa `init.sql`, recriando tabela e três exemplos | Volume vazio |

`docker volume ls` lista volumes e `docker volume inspect` mostra os metadados e o ponto de montagem do daemon. No Docker Desktop para macOS, esse `Mountpoint` fica dentro da máquina virtual Linux gerenciada pelo Docker e não é necessariamente um caminho diretamente navegável no macOS. `docker compose exec web ls -l /app/uploads` inspeciona os arquivos do volume a partir do contêiner que o utiliza.

Os resultados estão separados por finalidade: [`04-persistencia.txt`](evidencias/04-persistencia.txt) verifica preservação após recriar contêineres; [`05-remocao-volumes.txt`](evidencias/05-remocao-volumes.txt) demonstra o caso destrutivo com `down -v` e a reinicialização dos dados de exemplo. Assim, o segundo teste não deve ser interpretado como o procedimento normal de parada.

O script `init.sql` é inicialização, não migração: editar o arquivo e executar `up` não altera uma base persistida. Mudanças de esquema devem ser aplicadas com migrações incrementais e preservando os volumes; `down -v` é uma ação destrutiva e não deve ser usado para “atualizar” um banco com dados que se queira manter.

## 6. Segurança, escopo e operação

- O `.env` local não deve ser versionado; `.env.example` contém somente valores ilustrativos. As variáveis obrigatórias interrompem a renderização do Compose quando faltam.
- A API e o banco não publicam portas individualmente no host. O acesso HTTP passa pelo Nginx; Adminer permanece desligado no profile `debug` e vinculado a loopback.
- O usuário `app` tem permissão de escrita no diretório de fotos montado e executa Gunicorn sem root. A imagem e o contêiner Postgres ainda seguem seus próprios padrões de usuário.
- O digest torna a imagem reproduzível, mas não automaticamente atualizada. Revisar os digests é parte da manutenção de segurança.
- As rotas fornecidas não implementam autenticação/autorização nem validação aprofundada do tipo de arquivo. O limite de 10 MB no proxy é um limite de tamanho, não uma proteção completa para dados pessoais. Uma implantação real precisa tratar controles de acesso, TLS, retenção e segurança de backups.
- O Adminer e o Nginx estão fixados por digest no Compose. Isso torna cada referência de imagem específica ao conteúdo identificado, mas não instala automaticamente atualizações de segurança; os digests devem ser revisados e atualizados de forma deliberada.

Os detalhes dos bônus e seus comandos operacionais ficam em [`BONUS.md`](BONUS.md). A configuração e os limites de rede são ilustrados em [`arquitetura.md`](arquitetura.md); as execuções usadas para conferir as afirmações estão listadas em [`evidencias/README.md`](evidencias/README.md).
