# Reflexão e equivalência em nuvem

## Questões

### Q1. Por que o serviço web consegue se conectar ao banco usando o nome db em vez de um endereço IP? Quem resolve esse nome e o que mudaria se os contêineres fossem criados com docker run na rede bridge padrão?

Os serviços definidos em [`docker-compose.yml`](../docker-compose.yml) compartilham a bridge definida pelo usuário `rede_interna`, com DNS interno do Docker. A aplicação usa `DB_HOST=db`, e o DNS resolve esse nome para o endereço do serviço PostgreSQL sem colocar IPs efêmeros na configuração. Isso torna a comunicação mais legível e permite que o endereço seja administrado pelo Compose quando os contêineres são recriados. Já a bridge padrão usada diretamente por `docker run` não oferece a mesma resolução automática de nomes entre contêineres; nesse caso, deve-se criar uma rede definida pelo usuário ou configurar outra forma explícita de descoberta.

### Q2. Qual a diferença entre um volume nomeado e um bind mount? Justifique por que o init.sql foi montado como bind mount e os dados do banco como volume nomeado.

Um bind mount conecta um caminho conhecido do host a um caminho do contêiner; por isso `./db/init.sql` é montado diretamente e somente para leitura, facilitando revisão e manutenção do script junto ao código. Um volume nomeado é criado e gerenciado pelo Docker e não depende de escolher manualmente um diretório do host. No projeto, `dados_banco` guarda os arquivos mutáveis do PostgreSQL e `fotos` guarda os uploads em `/app/uploads`. Ambos sobrevivem à remoção dos contêineres, mas são apagados explicitamente por `docker compose down -v`.

### Q3. Se você alterar o init.sql (por exemplo, adicionando uma coluna) depois que o ambiente já subiu uma vez e executar docker compose up -d de novo, a mudança é aplicada? Por quê? Como aplicar a alteração sem perder dados?

Não. A imagem oficial do PostgreSQL executa scripts de `/docker-entrypoint-initdb.d` quando inicializa um diretório de dados vazio, não em toda subida do contêiner. Como `dados_banco` persiste após `docker compose down`, um novo `up` reutiliza o banco já inicializado e não reaplica `init.sql`. Essa característica evita recriar silenciosamente o banco em cada reinício, mas significa que alterações no script não migram dados existentes. Para evoluir o esquema, deve-se aplicar uma migração SQL incremental; remover o volume força uma inicialização limpa, mas elimina os dados e não é uma estratégia de migração.

### Q4. Por que depends_on sozinho, sem condition: service_healthy, não garante que a aplicação consiga se conectar ao banco na primeira tentativa?

`depends_on` sem condição ordena a criação ou inicialização dos contêineres, mas o estado “iniciado” não garante que o PostgreSQL já aceite consultas. O banco pode ainda estar criando arquivos, recuperando dados ou configurando o usuário inicial. O healthcheck de `db` executa `pg_isready`, e `web` usa `condition: service_healthy` para aguardar o resultado saudável. Depois, o próprio `web` tem healthcheck em `/health`, que verifica a aplicação e a conexão com o banco; o Nginx espera esse estado saudável antes de iniciar.

### Q5. A ordem das instruções COPY requirements.txt → RUN pip install → COPY app.py influencia o tempo de build. Explique usando o conceito de cache de camadas e cite o que acontece quando só o app.py muda.

Docker pode reutilizar cada camada quando as instruções e seus arquivos de entrada continuam iguais. O Dockerfile copia `requirements.txt` e instala os pacotes antes de copiar `app.py`, separando dependências de código-fonte. Assim, uma alteração apenas em `app.py` invalida a camada da cópia do código, mas permite reaproveitar a instalação do pip. Se o manifesto de dependências mudar, a instalação é refeita, pois o conjunto de pacotes da imagem também mudou.

### Q6. Por que a porta 5432 do banco não deve ser publicada no host? Considerando que as fotos podem mostrar documentos pessoais esquecidos (carteira de estudante, RG), relacione a sua configuração com a LGPD e proponha duas medidas adicionais de proteção.

A porta 5432 não é publicada no host porque somente `web` e, quando ativado, Adminer precisam acessar o banco pela rede Docker. Isso reduz a superfície de exposição e impede conexões diretas ao PostgreSQL pelas interfaces do host, embora não substitua controle de acesso entre serviços. As fotos também merecem atenção: a rota `/fotos/<nome>` fornecida pela API não exige autenticação, então quem conhece o nome do arquivo pode solicitá-lo. Duas medidas adicionais seriam implementar autenticação/autorização com privilégio mínimo, inclusive para leitura de fotos, e servir o tráfego externo por HTTPS. Também seriam importantes minimizar a coleta, definir retenção e exclusão, validar tipo e tamanho do arquivo e proteger backups; esses controles não fazem parte da API entregue.

### Q7. Tente executar `docker compose up -d --scale web=3`. O que acontece e por quê? O que seria necessário para rodar três réplicas da aplicação? As réplicas poderiam compartilhar o volume fotos com segurança?

Como `web` não declara `ports`, o Compose não precisa reservar uma porta fixa do host para cada réplica. O teste registrado em [`09-escalabilidade.txt`](evidencias/09-escalabilidade.txt) criou/confirmou três réplicas saudáveis, sem conflito de porta. Porém, dez chamadas de saúde via Nginx foram respondidas por `web-1`; o teste não observou tráfego nas outras duas. Isso é compatível com a configuração atual de [`nginx/default.conf`](../nginx/default.conf), que encaminha para `web:5000` sem definir explicitamente um upstream de balanceamento com descoberta dinâmica e verificação de saúde. Portanto, escalar o serviço criou réplicas, mas não demonstrou distribuição de carga. Para uma implantação escalável, é necessário configurar um proxy/balanceador que descubra endpoints saudáveis e validar a distribuição do tráfego.

No mesmo host Docker, as instâncias poderiam montar o volume `fotos` compartilhado; os nomes UUID reduzem colisões simples de nome, mas isso não transforma o volume local em armazenamento distribuído nem resolve alta disponibilidade. Réplicas em hosts distintos precisariam de storage compartilhado compatível, como armazenamento de objetos, e de regras explícitas de acesso e retenção. O banco também receberia conexões concorrentes das réplicas e precisaria ser dimensionado e monitorado.

## 5.1 Equivalência do ambiente local

O Compose atende ao ambiente de desenvolvimento e demonstra os componentes, mas não equivale automaticamente a uma arquitetura gerenciada de produção. A tabela associa cada responsabilidade local a serviços possíveis; é uma correspondência conceitual, não uma afirmação de que todos têm exatamente o mesmo modelo operacional.

| Responsabilidade local | Microsoft Azure | AWS | Google Cloud |
| --- | --- | --- | --- |
| Executar contêineres da aplicação | Azure Container Apps | Amazon ECS com AWS Fargate | Cloud Run |
| Banco PostgreSQL e persistência relacional | Azure Database for PostgreSQL Flexible Server | Amazon RDS for PostgreSQL | Cloud SQL for PostgreSQL |
| Fotos e arquivos de usuário | Azure Blob Storage | Amazon S3 | Cloud Storage |
| Rede privada e regras de tráfego | Azure Virtual Network (VNet) e integração de rede do serviço de contêineres | Amazon VPC, sub-redes privadas e security groups | Virtual Private Cloud (VPC) e regras de firewall |
| Senhas e configuração secreta | Azure Key Vault | AWS Secrets Manager | Secret Manager |
| Armazenar imagens de contêiner | Azure Container Registry (ACR) | Amazon Elastic Container Registry (ECR) | Artifact Registry |
| Entrada HTTP, TLS e distribuição | Azure Front Door ou Application Gateway | Application Load Balancer | Cloud Load Balancing |
| Logs e métricas operacionais | Azure Monitor | Amazon CloudWatch | Cloud Logging e Cloud Monitoring |

Na migração, fotos são provavelmente o componente que mais exige adaptação: a aplicação atual grava em `/app/uploads`, enquanto um serviço de objetos requer operações por API/SDK, identidade de serviço, política de acesso e possivelmente URLs assinadas. O banco também passa a usar endpoint e credenciais gerenciadas; variáveis de configuração podem continuar desacoplando a aplicação do endereço concreto, mas segredos devem vir de um cofre, não de um `.env` distribuído. A imagem construída localmente precisaria ser publicada em um registry e referenciada pelo serviço de contêineres. Rede, TLS, backups, disponibilidade, retenção, observabilidade e custos dependem do provedor e do nível de serviço escolhido. A escolha entre serviços de contêiner gerenciados também deve considerar requisitos de execução contínua, escalabilidade, observabilidade e compatibilidade com o processo da aplicação.
