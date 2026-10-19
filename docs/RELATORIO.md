# Relatório técnico

## Dockerfile da aplicação

O `app/Dockerfile` parte de `python:3.12-slim`, uma imagem oficial enxuta com o runtime exigido. `PYTHONDONTWRITEBYTECODE` evita arquivos `.pyc` no contêiner e `PYTHONUNBUFFERED` envia os logs sem atraso. `WORKDIR /app` define o diretório de trabalho.

`COPY requirements.txt .` e `RUN pip install --no-cache-dir -r requirements.txt` vêm antes da cópia do código. Assim, uma alteração isolada em `app.py` não invalida a camada de dependências e o pip não deixa cache na imagem. Na mesma etapa são criados o grupo e o usuário de sistema `app`, o diretório `/app/uploads` e sua permissão de escrita. `COPY --chown=app:app app.py .` copia o código com proprietário não privilegiado. `USER app` garante que o processo não execute como root. `EXPOSE 5000` documenta a porta interna e o `CMD` em formato exec inicia Gunicorn em `0.0.0.0:5000`, tornando a aplicação acessível aos outros contêineres.

O `app/.dockerignore` exclui `.env`, `__pycache__/`, arquivos `.pyc`/`.pyo` e `.git/` do contexto de build. Isso reduz o contexto e evita incluir configuração local e artefatos Python.

As imagens auxiliares Nginx e Adminer são fixadas por digest para que o Compose use o mesmo conteúdo de imagem entre execuções. Para receber correções de segurança e atualizações, os digests devem ser revisados e atualizados deliberadamente.

## Blocos do Docker Compose

- **`db`**: usa a imagem oficial `postgres:16-alpine`. As variáveis `POSTGRES_DB`, `POSTGRES_USER` e `POSTGRES_PASSWORD` vêm do `.env`, sem valores de senha no YAML. O volume `dados_banco` guarda os arquivos do PostgreSQL em `/var/lib/postgresql/data`. `db/init.sql` é montado como bind mount somente leitura no diretório de inicialização do entrypoint oficial; esse script é executado quando o diretório de dados ainda está vazio.
- **Healthcheck**: `pg_isready` verifica se o banco já aceita conexões, com intervalos, timeout, tentativas e período inicial definidos. O Compose usa esse estado para liberar a aplicação.
- **`web`**: constrói a imagem a partir de `./app`, passa ao código as variáveis de conexão e define `/app/uploads` como diretório de fotos. Não publica portas no host. O volume `fotos` mantém os arquivos fora do ciclo de vida do contêiner. Seu healthcheck consulta `/health` e confirma que a aplicação também consegue acessar o banco.
- **`nginx`**: usa a imagem oficial `nginx:alpine` fixada por digest, monta a configuração do proxy como somente leitura e é o único serviço da aplicação com porta publicada (`8080:80`). Encaminha as requisições HTTP para `web:5000`, encaminha os cabeçalhos de host e IP do cliente e permite uploads de até 10 MB (`client_max_body_size 10M`), acima do limite padrão do Nginx.
- **`adminer`**: usa a imagem oficial Adminer fixada por digest no profile `debug`, aponta para o servidor `db` e publica `8081` somente no loopback do host. O serviço não inicia na configuração padrão.
- **`depends_on`**: a condição `service_healthy` impede a inicialização do `web` antes de o healthcheck do banco passar.
- **Rede**: `rede_interna` é uma bridge definida pelo usuário, compartilhada pelos serviços; `db` tem um alias explícito para resolução pelo DNS interno do Docker. O banco não publica a porta 5432 e `web` não publica a porta 5000; somente o Nginx publica a porta HTTP `8080` no host.
- **Reinício e volumes**: `restart: unless-stopped` mantém os serviços em execução após falhas/reinícios, salvo parada explícita. `dados_banco` e `fotos` são volumes nomeados, declarados na raiz do Compose.

## Caminho de um `POST /itens` com foto

1. `curl` envia `multipart/form-data` para `http://localhost:8080/itens`; a publicação `8080:80` entrega a requisição ao contêiner Nginx.
2. Nginx encaminha o método, o caminho, o corpo e os cabeçalhos ao serviço `web:5000` pela `rede_interna`.
3. Gunicorn recebe a requisição e encaminha `/itens` à rota Flask `cadastrar`.
4. Flask lê `descricao`, `local` e o arquivo `foto`, gera um nome aleatório baseado em UUID e salva o arquivo em `/app/uploads`.
5. `/app/uploads` é o ponto de montagem do volume nomeado `fotos`; portanto, o arquivo permanece no volume mesmo se o contêiner `web` for removido.
6. A aplicação conecta-se ao host `db`, porta 5432, usando as credenciais de ambiente. O DNS da `rede_interna` resolve `db` para o contêiner PostgreSQL.
7. A instrução `INSERT` grava os campos e o nome do arquivo na tabela `itens`; os dados da tabela ficam no volume `dados_banco`.
8. A API retorna HTTP 201 com o identificador do item e o nome da foto. `GET /itens` fornece a URL `/fotos/<nome>`, servida pelo endpoint de arquivos a partir de `/app/uploads` e entregue ao cliente pelo Nginx.

O arquivo e o registro são gravados em armazenamentos separados, sem uma transação distribuída; uma falha entre a gravação do arquivo e a inserção no banco pode deixar um arquivo sem registro.

## Ciclo de vida dos volumes

- `docker compose down` remove os contêineres e a rede, mas mantém os volumes nomeados. Depois de `docker compose up -d`, os registros e fotos continuam disponíveis.
- `docker volume ls` lista volumes; `docker volume inspect <volume>` mostra metadados e o ponto de montagem gerenciado pelo Docker. O conteúdo de fotos pode ser verificado com `docker compose exec web ls -l /app/uploads`.
- No Docker Desktop para macOS, o `Mountpoint` exibido pertence ao daemon dentro da máquina virtual Linux do Docker; não é um caminho diretamente navegável no filesystem do macOS.
- `docker compose down -v` remove também os volumes declarados pelo projeto. Itens cadastrados e fotos são apagados. No próximo `up`, o volume vazio do PostgreSQL faz o entrypoint executar `init.sql` novamente, recriando a tabela e os três itens de exemplo.

O `init.sql` não é uma ferramenta de migração: ele roda somente na inicialização de um diretório de dados vazio. Para alterar uma tabela existente, deve-se aplicar uma migração SQL versionada ao banco, preservando os volumes.

## Extras: proxy, backup e painel de banco

O Nginx é o único serviço que publica a porta 8080 no host; o `web` permanece acessível apenas pela rede Compose. O serviço Adminer é condicionado ao profile `debug`, disponível em `127.0.0.1:8081` e conectado à rede interna para acessar o banco sem publicar a porta PostgreSQL.

O backup das fotos usa um contêiner descartável com o volume ativo montado somente para leitura e grava um arquivo `.tar.gz` no host. A restauração extrai o arquivo para um novo volume nomeado e define os arquivos como pertencentes ao UID/GID de `app`, permitindo seu uso pela aplicação sem modificar o volume original. O roteiro e as saídas da verificação estão em [README.md](../README.md) e nas evidências.
