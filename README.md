# Achados & Perdidos

API Flask para consulta e cadastro de objetos perdidos, executada com Gunicorn e PostgreSQL em contêineres Docker. O Compose mantém os serviços em uma rede bridge definida pelo projeto, publica a aplicação somente pelo Nginx e persiste os dados do banco e as fotos em volumes nomeados.

## Pré-requisitos

- Docker Engine com o plugin Docker Compose.
- `curl` (ou `curl.exe` no PowerShell do Windows).

Não é necessário instalar Python ou PostgreSQL no host.

## Configuração e inicialização

Na raiz do projeto, crie o arquivo local de configuração a partir do modelo:

```sh
cp .env.example .env
```

Edite `.env` e substitua `POSTGRES_PASSWORD` por uma senha local forte (valores com caracteres especiais como `$` podem ser delimitados por aspas simples). O `.env` não deve ser versionado nem incluído no pacote entregue.

Suba a aplicação e o banco por trás do proxy:

```sh
docker compose up -d --build
docker compose ps
```

Espere até `db` e `web` aparecerem como `healthy` e `nginx` como `running`. A base da API é `http://localhost:8080`; ela não tem página inicial em `/`. Para conferir no navegador, abra `http://localhost:8080/health`. Apenas o Nginx publica uma porta no host; os contêineres participam da `rede_interna`, e o PostgreSQL não publica a porta 5432.

## Testes rápidos

```sh
curl http://localhost:8080/health
curl http://localhost:8080/itens
curl "http://localhost:8080/itens?local=biblio"
curl http://localhost:8080/estatisticas
```

Para cadastrar um objeto com uma foto JPG ou PNG existente no computador:

```sh
curl -F "descricao=Oculos de grau" \
     -F "local=Bloco B" \
     -F "foto=@/caminho/para/oculos.jpg" \
     http://localhost:8080/itens
```

A resposta inclui o nome gerado para a foto. Consulte `GET /itens` para obter a URL e abra `http://localhost:8080/fotos/<nome>` no navegador. O Nginx aceita uploads de até 10 MB por requisição. Para ver os logs recentes da aplicação:

```sh
docker compose logs --tail 20 web
```

## Rotas da API

| Método | Rota | Descrição |
| --- | --- | --- |
| `GET` | `/health` | Verifica a conexão com o PostgreSQL. |
| `GET` | `/itens` | Lista os objetos cadastrados. |
| `GET` | `/itens?local=texto` | Filtra objetos pelo local (busca parcial, sem diferenciar maiúsculas de minúsculas). |
| `POST` | `/itens` | Cadastra `descricao` e `local` como campos multipart/form-data; `foto` é opcional. |
| `GET` | `/fotos/<nome>` | Devolve a foto armazenada no volume de uploads. |
| `GET` | `/estatisticas` | Informa o número de itens e de arquivos no volume de fotos. |

## Arquitetura

O diagrama completo, incluindo os limites da rede, as portas publicadas, os volumes e os bind mounts, está em [docs/arquitetura.md](docs/arquitetura.md). O host acessa a API pelo Nginx na porta `8080`; `web` e `db` não publicam portas no host. O Adminer só fica disponível em `127.0.0.1:8081` quando ativado pelo profile `debug`.

## Painel do banco (opcional)

O Adminer está no profile opcional `debug`, então não é iniciado no uso normal. Para ativá-lo quando precisar administrar o banco:

```sh
docker compose --profile debug up -d
```

Abra `http://localhost:8081` e use `db` como servidor, o usuário, a senha e o banco definidos no `.env`, e PostgreSQL como sistema. A porta 8081 é publicada somente em `127.0.0.1`; não a exponha a uma rede pública. Para parar o painel sem derrubar os outros serviços:

```sh
docker compose --profile debug stop adminer
```

O guia de [desafios bônus](docs/BONUS.md) explica o profile, os campos de conexão e a restrição da porta ao loopback.

## Backup e restauração das fotos

O procedimento completo, incluindo inspeção do arquivo e restauração em um volume isolado com verificação de hashes, está no [guia de desafios bônus](docs/BONUS.md). O comando mínimo para criar uma cópia do volume ativo é:

```sh
docker run --rm \
  -v achados-perdidos_fotos:/data:ro \
  -v "$PWD/docs/evidencias:/backup" \
  alpine:3.21 \
  tar -czf /backup/fotos-backup.tar.gz -C /data .
```

O exemplo usa o nome padrão `achados-perdidos_fotos`; se o diretório ou o nome do projeto Compose for diferente, confira `docker volume ls` e substitua pelo volume correto. O arquivo de backup incluído na entrega contém somente imagens artificiais de teste.

## Persistência, parada e remoção

Para parar os contêineres sem removê-los e iniciá-los novamente:

```sh
docker compose stop
docker compose start
```

`docker compose down` remove os contêineres e a rede, mas preserva `dados_banco` e `fotos`. Ao executar `docker compose up -d` novamente, os itens cadastrados e suas fotos permanecem disponíveis.

Para destruir também os dados persistidos (ação irreversível):

```sh
docker compose down -v
```

Ao subir de novo, o PostgreSQL cria um banco vazio e executa `db/init.sql`, que insere os três objetos de exemplo. Cadastros e fotos anteriores serão perdidos. Alterar `init.sql` não modifica um banco já inicializado; alterações de esquema devem ser aplicadas por migrações.

## Documentação complementar

- [Relatório técnico](docs/RELATORIO.md): decisões do Dockerfile e do Compose, fluxo de cadastro com foto e ciclo de vida dos volumes.
- [Desafios bônus](docs/BONUS.md): explicação e comandos para Nginx, backup/restauração e Adminer.
- [Diagrama de arquitetura](docs/arquitetura.md): rede, serviços, portas, volumes e bind mounts.
- [Reflexão e equivalência em nuvem](docs/REFLEXAO.md): respostas às questões e opções de serviços gerenciados.
- [Índice de evidências](docs/evidencias/README.md): mapa entre os requisitos e as saídas de teste correspondentes.
- [Backup de teste das fotos](docs/evidencias/fotos-backup.tar.gz): arquivo usado para demonstrar restauração em volume separado.
