# Reflexão e equivalência em nuvem

## Questões

### Q1. Resolução do nome `db`

Os serviços definidos em `docker-compose.yml` compartilham a rede bridge `rede_interna`, criada pelo Compose com DNS interno. Esse DNS resolve o nome de serviço `db` para o endereço atual do contêiner PostgreSQL, sem fixar IP no código. Na bridge padrão usada diretamente por `docker run`, a resolução automática de nomes entre contêineres não é equivalente; seria preciso criar uma rede definida pelo usuário ou configurar links/endereços explicitamente.

### Q2. Volume nomeado e bind mount

Um bind mount aponta para um caminho específico do host; por isso `./db/init.sql` é montado diretamente e somente para leitura no contêiner, facilitando a manutenção do script no repositório. Um volume nomeado é criado e gerenciado pelo Docker, não depende de um caminho escolhido manualmente no host e sobrevive à remoção dos contêineres. Ele é apropriado para os dados mutáveis de `dados_banco` e para as fotos em `fotos`.

### Q3. Alteração do `init.sql`

Não. O entrypoint do PostgreSQL executa os scripts de `/docker-entrypoint-initdb.d` apenas quando inicializa um diretório de dados vazio. Como `dados_banco` persiste após `down`, um novo `up` não reaplica o script. A forma segura de alterar o esquema é escrever e executar uma migração SQL incremental; apagar o volume reaplica a inicialização, mas também apaga todos os dados existentes.

### Q4. `depends_on` e prontidão

`depends_on` sem condição ordena a criação/inicialização dos contêineres, mas “iniciado” não significa que o PostgreSQL já esteja aceitando conexões. O processo do banco ainda pode estar configurando arquivos ou recuperando dados. O healthcheck com `pg_isready` e `condition: service_healthy` faz o Compose aguardar a prontidão do banco antes de iniciar `web`.

### Q5. Cache de camadas

Docker reutiliza a camada de instalação enquanto `requirements.txt` e as instruções anteriores não mudarem. Ao copiar dependências antes de `app.py`, uma alteração somente no código invalida a camada de cópia da aplicação, mas reaproveita a instalação do pip. Se código e dependências fossem copiados juntos antes do `pip install`, qualquer mudança em `app.py` poderia obrigar a reinstalar todas as dependências.

### Q6. Porta do banco, LGPD e medidas adicionais

A porta 5432 não é publicada no host porque apenas `web` precisa acessar o banco, pela rede privada; isso reduz a superfície de ataque e impede conexões externas diretas. Fotos podem conter dados pessoais, então o projeto deve limitar coleta e retenção e controlar quem pode consultá-las. Duas medidas adicionais são autenticação/autorização com privilégio mínimo na API e HTTPS entre usuários e a aplicação. Também são recomendáveis validar tipo e tamanho de arquivo, restringir acesso aos volumes, proteger backups e definir um processo de exclusão de dados.

### Q7. Escala do `web`

Com o proxy reverso Nginx da tarefa bônus, `web` não publica porta no host, então as réplicas não disputam a porta 8080; o acesso externo continua passando pelo proxy. Para uma escala confiável, o balanceador precisa distribuir requisições entre as réplicas e detectar falhas, e o proxy deve acompanhar as mudanças de DNS/endpoints ao escalar. As réplicas podem montar o mesmo volume local `fotos` no mesmo host, mas isso não fornece armazenamento distribuído nem torna seguro escalar entre hosts. Em nuvem, armazenamento de objetos ou um filesystem compartilhado, permissões e ciclo de vida de fotos devem ser projetados explicitamente.

## 5.1 Equivalência do ambiente local

| Peça do ambiente local | Microsoft Azure | AWS | Google Cloud |
| --- | --- | --- | --- |
| Contêiner `web` | Azure Container Apps | Amazon ECS com AWS Fargate | Cloud Run |
| Contêiner `db` + volume `dados_banco` | Azure Database for PostgreSQL Flexible Server | Amazon RDS for PostgreSQL | Cloud SQL for PostgreSQL |
| Volume `fotos` | Azure Blob Storage | Amazon S3 | Cloud Storage |
| `rede_interna` | Azure Virtual Network (VNet) e integração de rede do serviço de contêineres | Amazon VPC e sub-redes privadas | Virtual Private Cloud (VPC) |
| Arquivo `.env` (segredos) | Azure Key Vault | AWS Secrets Manager | Secret Manager |
| Imagem construída localmente | Azure Container Registry (ACR) | Amazon Elastic Container Registry (ECR) | Artifact Registry |

Na migração, o armazenamento de fotos tende a exigir a maior mudança de aplicação: o caminho local `/app/uploads` deixaria de ser a fronteira correta e os arquivos normalmente passariam para um serviço de objetos com autenticação e URLs controladas. A conexão do banco e os segredos também mudariam para endpoints e serviços gerenciados, mas podem continuar desacoplados por variáveis de ambiente e configuração de deploy. As opções da tabela são equivalentes conceituais; rede, identidade, backup, disponibilidade e custos precisam ser definidos para o provedor escolhido.
