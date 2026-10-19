# Evidências

Este diretório guarda saídas de comandos e testes executados no ambiente Docker do projeto. A tabela permite localizar uma evidência a partir da etapa da atividade, sem exigir que cada comando fique em um arquivo separado.

| Etapa ou requisito | Evidência | O que permite conferir |
| --- | --- | --- |
| Build e inicialização | [`01-subida.txt`](01-subida.txt) | Saída de `docker compose up -d --build` e estado inicial dos serviços. |
| Rotas e comportamento da API | [`02-testes-funcionais.txt`](02-testes-funcionais.txt) | Estado saudável, listagem, filtro por local, estatísticas e cadastro multipart com foto. |
| Inspeção da implantação | [`03-inspecoes.txt`](03-inspecoes.txt) | Estado dos serviços, usuário não-root, volumes, rede, logs e tamanho da imagem. |
| Persistência após recriar contêineres | [`04-persistencia.txt`](04-persistencia.txt) | Estatísticas e leitura das fotos antes e depois de `docker compose down` seguido de `up`. |
| Comportamento de `down -v` e inicialização | [`05-remocao-volumes.txt`](05-remocao-volumes.txt) | Remoção explícita dos volumes e reinicialização dos três itens de exemplo pelo `init.sql`. É um teste destrutivo feito no ambiente de demonstração. |
| Proxy e painel opcional | [`06-extras-nginx-adminer.txt`](06-extras-nginx-adminer.txt) | Nginx como entrada HTTP, ausência de porta publicada pelo `web` e Adminer ativado pelo profile `debug`. |
| Backup de fotos | [`07-backup-restauracao.txt`](07-backup-restauracao.txt) | Criação do TAR.GZ, restauração em volume independente e comparação dos arquivos por hash. |
| Limite de upload do proxy | [`08-limite-upload-nginx.txt`](08-limite-upload-nginx.txt) | Upload real de PNG válido com mais de 1 MB por `POST /itens`, resposta HTTP 201, download HTTP 200 e comparação byte a byte. |
| Escala do serviço web | [`09-escalabilidade.txt`](09-escalabilidade.txt) | Três réplicas saudáveis sem portas publicadas individualmente; dez chamadas via Nginx chegaram à réplica `web-1`, demonstrando que a configuração não distribuiu tráfego nesse teste. |
| Artefato de restauração | [`fotos-backup.tar.gz`](fotos-backup.tar.gz) | Backup incluído para reproduzir a restauração em outro volume; contém somente imagens artificiais de teste. |

## Como interpretar

- As evidências são registros de uma execução de teste, não uma garantia de que serviços estejam ativos agora. Para reproduzir, siga os comandos do [README](../../README.md) e os procedimentos em [`../BONUS.md`](../BONUS.md).
- Os nomes de volume usados nos exemplos seguem o nome do projeto Compose desta entrega. Compose pode prefixá-los com outro nome se a pasta ou `COMPOSE_PROJECT_NAME` mudar; confira `docker volume ls` antes de copiar comandos de backup.
- O arquivo de backup é um artefato de teste e não deve ser tratado como cópia de dados reais. Backups de produção precisam de controle de acesso, retenção e armazenamento apropriado.

Os uploads de teste usaram imagens PNG artificiais, sem dados pessoais.
