# Evidências

- `01-subida.txt` — saída do build e da inicialização com `docker compose up -d --build`.
- `02-testes-funcionais.txt` — estado saudável, listagem, filtro, estatísticas e cadastro com foto pela API.
- `03-inspecoes.txt` — estado final dos serviços, usuário não-root, volumes, rede, logs e tamanho da imagem.
- `04-persistencia.txt` — estatísticas e leitura das duas fotos antes e depois de `docker compose down` seguido de `up`.
- `05-remocao-volumes.txt` — remoção dos volumes com `down -v` e reinicialização dos três itens de exemplo pelo `init.sql`.
- `06-extras-nginx-adminer.txt` — proxy reverso, ausência de porta publicada pelo `web` e interface Adminer ativada pelo profile `debug`.
- `07-backup-restauracao.txt` — criação do arquivo TAR.GZ, restauração em volume independente e comparação dos arquivos.
- `08-limite-upload-nginx.txt` — requisição multipart de 1,5 MB encaminhada ao Flask sem bloqueio pelo limite padrão de 1 MB do Nginx.
- `fotos-backup.tar.gz` — backup gerado no teste; contém apenas as imagens PNG mínimas de teste, sem dados pessoais.

Os uploads de teste usaram uma imagem PNG mínima temporária; ela não contém dados pessoais nem foi incluída no repositório.
