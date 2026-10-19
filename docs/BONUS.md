# Desafios bônus

Os três extras da seção 6 foram implementados e testados: proxy reverso Nginx (+0,4), backup e restauração do volume de fotos (+0,3) e painel Adminer controlado pelo profile `debug` (+0,3). Os comandos devem ser executados na raiz do projeto, onde está `docker-compose.yml`.

## 1. Proxy reverso Nginx (+0,4)

### Configuração

O serviço `nginx` usa a imagem oficial fixada por digest no [Compose](../docker-compose.yml). A configuração está em [`nginx/default.conf`](../nginx/default.conf) e é montada no contêiner em `/etc/nginx/conf.d/default.conf` como somente leitura.

O caminho da requisição é:

```text
Navegador / curl
    → host:8080
    → nginx:80
    → web:5000 (rede Docker)
    → Flask / Gunicorn
```

O Nginx é o único serviço que publica a porta principal no host (`8080:80`). O serviço `web` continua escutando em `0.0.0.0:5000` dentro do contêiner, mas não tem `ports`; o PostgreSQL também não publica a porta 5432. Os cabeçalhos `Host`, `X-Real-IP`, `X-Forwarded-For` e `X-Forwarded-Proto` são encaminhados ao backend.

`client_max_body_size 10M` aumenta o limite padrão do Nginx para permitir arquivos maiores. A aplicação continua responsável pelo processamento do formulário; a configuração do proxy não substitui validação de arquivo ou autenticação.

### Testes e evidências

- [`01-subida.txt`](evidencias/01-subida.txt) registra a inicialização da versão com proxy e a validação de sintaxe do Nginx.
- [`02-testes-funcionais.txt`](evidencias/02-testes-funcionais.txt) registra chamadas GET, cadastro multipart com foto e leitura das imagens através do proxy.
- [`03-inspecoes.txt`](evidencias/03-inspecoes.txt) mostra que `nginx` é o único dos três serviços principais com porta publicada e que `web` executa como usuário `app`.
- [`08-limite-upload-nginx.txt`](evidencias/08-limite-upload-nginx.txt) mostra um POST real para `/itens` com PNG válido de 1.403.937 bytes: HTTP 201 no cadastro, HTTP 200 no download e comparação byte a byte com o arquivo enviado.

O teste de upload grande usa dados sintéticos, não uma fotografia pessoal.

## 2. Backup e restauração de `fotos` (+0,3)

Um contêiner Alpine descartável lê o volume de origem como somente leitura e escreve o arquivo compactado na pasta `docs/evidencias/`. O nome completo do volume criado pelo Compose normalmente segue `<projeto>_fotos`. Confira com `docker volume ls` e ajuste o nome abaixo se a pasta ou o nome do projeto Compose for diferente.

### Criar o backup

```sh
docker volume ls
docker run --rm \
  -v achados-perdidos_fotos:/data:ro \
  -v "$PWD/docs/evidencias:/backup" \
  alpine:3.21 \
  tar -czf /backup/fotos-backup.tar.gz -C /data .
```

O volume é montado como `:ro` para que o processo de backup não altere os arquivos originais. O diretório de destino é um bind mount do host, permitindo guardar e copiar o arquivo TAR.GZ sem depender do ciclo de vida do volume.

Confira o conteúdo do arquivo:

```sh
docker run --rm \
  -v "$PWD/docs/evidencias:/backup:ro" \
  alpine:3.21 \
  tar -tzvf /backup/fotos-backup.tar.gz
```

### Restaurar em um volume novo

O exemplo restaura em `fotos_restauradas`, sem sobrescrever o volume usado pela aplicação. O UID/GID do usuário `app` é consultado na imagem para que os arquivos restaurados tenham permissões compatíveis com o processo não-root.

```sh
docker volume create fotos_restauradas
APP_UID="$(docker compose exec -T web id -u app)"
APP_GID="$(docker compose exec -T web id -g app)"
docker run --rm \
  -e APP_UID="$APP_UID" -e APP_GID="$APP_GID" \
  -v fotos_restauradas:/data \
  -v "$PWD/docs/evidencias:/backup:ro" \
  alpine:3.21 \
  sh -c 'tar -xzf /backup/fotos-backup.tar.gz -C /data && chown -R "$APP_UID:$APP_GID" /data'
docker run --rm -v fotos_restauradas:/data alpine:3.21 ls -ln /data
```

Para verificar que nada mudou, compare os SHA-256 dos arquivos dos dois volumes:

```sh
docker run --rm -v achados-perdidos_fotos:/data:ro alpine:3.21 \
  sh -c 'cd /data && sha256sum * | sort'
docker run --rm -v fotos_restauradas:/data:ro alpine:3.21 \
  sh -c 'cd /data && sha256sum * | sort'
```

Depois de conferir a restauração, remova somente o volume temporário:

```sh
docker volume rm fotos_restauradas
```

### Resultado registrado

[`07-backup-restauracao.txt`](evidencias/07-backup-restauracao.txt) registra as estatísticas antes do backup, a lista de arquivos, a extração no volume novo e os hashes idênticos entre origem e restauração. O arquivo de teste [`fotos-backup.tar.gz`](evidencias/fotos-backup.tar.gz) acompanha a entrega e contém apenas imagens sintéticas de teste. Ele serve para demonstrar o procedimento, não é um backup de dados reais de usuários.

## 3. Adminer no profile `debug` (+0,3)

O serviço Adminer está definido no Compose com `profiles: ["debug"]`. Um profile é uma chave de ativação do Docker Compose: serviços associados ao profile não são iniciados pelo `docker compose up` normal. Isso evita deixar o painel de administração ativo quando não é necessário.

Inicie o painel:

```sh
docker compose --profile debug up -d
docker compose --profile debug ps
```

Abra <http://localhost:8081> e preencha o formulário:

| Campo | Valor |
| --- | --- |
| Sistema | PostgreSQL |
| Servidor | `db` |
| Usuário | valor de `POSTGRES_USER` no `.env` |
| Senha | valor de `POSTGRES_PASSWORD` no `.env` |
| Banco de dados | valor de `POSTGRES_DB` no `.env` |

`db` funciona como hostname porque Adminer e PostgreSQL participam da mesma `rede_interna`, cujo DNS resolve nomes de serviço. A porta de Adminer fica vinculada a `127.0.0.1:8081`, portanto não é publicada em todas as interfaces do host. A porta 5432 do PostgreSQL continua sem publicação.

Para parar apenas o painel:

```sh
docker compose --profile debug stop adminer
```

O Adminer é uma ferramenta de administração, não uma rota da API nem parte necessária do fluxo normal. Mantenha-o desligado quando não estiver usando.

### Resultado registrado

[`06-extras-nginx-adminer.txt`](evidencias/06-extras-nginx-adminer.txt) mostra o serviço ausente na configuração normal, o profile ativado, a interface respondendo HTTP 200 em `localhost:8081` e o painel parado ao final do teste.

## Referências

- Configuração dos serviços: [`docker-compose.yml`](../docker-compose.yml).
- Configuração do proxy: [`nginx/default.conf`](../nginx/default.conf).
- Arquitetura, incluindo portas e bind mounts: [`arquitetura.md`](arquitetura.md).
- Saídas de execução: [`evidencias/README.md`](evidencias/README.md).
