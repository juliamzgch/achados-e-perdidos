# Evidências

- `01-subida.txt` — saída do build e da inicialização com `docker compose up -d --build`.
- `02-testes-funcionais.txt` — estado saudável, listagem, filtro, estatísticas e cadastro com foto pela API.
- `03-inspecoes.txt` — estado final dos serviços, usuário não-root, volumes, rede, logs e tamanho da imagem.
- `04-persistencia.txt` — estatísticas e leitura das duas fotos antes e depois de `docker compose down` seguido de `up`.
- `05-remocao-volumes.txt` — remoção dos volumes com `down -v` e reinicialização dos três itens de exemplo pelo `init.sql`.
- `06-extras-nginx-adminer.txt` — proxy reverso, ausência de porta publicada pelo `web` e interface Adminer ativada pelo profile `debug`.
- `07-backup-restauracao.txt` — criação do arquivo TAR.GZ, restauração em volume independente e comparação dos arquivos.
- `08-limite-upload-nginx.txt` — upload multipart real de uma imagem PNG com mais de 1 MB, cadastrado pela API através do Nginx.
- `fotos-backup.tar.gz` — backup atualizado do volume; inclui imagens PNG artificiais de teste, sem dados pessoais.

Os uploads de teste usaram imagens PNG artificiais, sem dados pessoais.
