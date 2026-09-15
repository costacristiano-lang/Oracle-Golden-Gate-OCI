# OCI GoldenGate: Oracle Database para Oracle Database

Guia genérico para configurar replicação Oracle-to-Oracle com OCI GoldenGate Data Replication.

> Este documento não depende de nomes, IPs ou compartments específicos. Substitua todos os valores entre `<...>` pelos dados do ambiente.

## Arquitetura

```text
Oracle Database origem
        |
        |  Extract + trail
        v
OCI GoldenGate Deployment
        |
        |  Distribution Path
        v
Oracle Database destino
        |
        v
     Replicat
```

Um único Deployment pode atender origem e destino em ambientes simples. Para produção, avalie Deployments separados conforme volume, isolamento, disponibilidade e operação.

## 1. Pré-requisitos

Antes de começar, reúna:

- Versão e patch level dos bancos origem e destino.
- Tipo de banco: non-CDB, CDB/PDB ou RAC.
- Host ou SCAN, porta do listener e service name de cada banco.
- CIDRs da VCN, subnets disponíveis, NSGs e rota até os bancos.
- Usuário Oracle dedicado ao GoldenGate, por exemplo `GGADMIN`.
- Wallets, caso a conexão use TCPS/TLS.
- Schemas, tabelas, chaves e estratégia de carga inicial.

Confirme na matriz do OCI GoldenGate se a versão do Deployment escolhida suporta **ambos** os bancos. Para RAC, informe o SCAN FQDN em vez do IP de um nó.

## 2. IAM

Este roteiro usa o grupo padrão `Administrators`, sem o prefixo do Identity Domain.

Crie ou confirme a policy administrativa:

```text
allow group Administrators to manage all-resources in tenancy
```

O GoldenGate também precisa das policies abaixo para trabalhar com Vault, chaves e IAM Identity Domains:

```text
allow service goldengate to use keys in tenancy
allow service goldengate to use vaults in tenancy

allow service goldengate to {idcs_user_viewer, domain_resources_viewer} in tenancy
```

### Dynamic Group para secrets

Quando connections usam password secrets ou wallet secrets, o Deployment precisa ler os bundles de secrets.

1. Acesse **Identity & Security → Dynamic Groups → Create Dynamic Group**.
2. Defina um nome, por exemplo `dg-ogg-deployments`.
3. Use a regra abaixo, substituindo o OCID do compartment do Deployment:

```text
ALL {resource.type = 'goldengatedeployment', resource.compartment.id = '<compartment-ocid>'}
```

4. Crie a policy:

```text
allow dynamic-group dg-ogg-deployments to read secret-bundles in tenancy
```

## 3. Rede

Crie ou reserve uma subnet **privada**, sem sobreposição com outras subnets e contida no CIDR da VCN. Exemplo:

```text
VCN:                  10.0.0.0/16
Subnet do GoldenGate: 10.0.10.0/24
```

Configure NSGs ou Security Lists conforme a topologia:

| Origem | Destino | Protocolo/porta | Finalidade |
|---|---|---:|---|
| Bastion, VPN ou rede administrativa | Endpoint GoldenGate | TCP 443 | Console e Admin Client |
| Ingress IPs do GoldenGate | Banco origem | TCP 1521 ou porta real | Capture / conexão Oracle |
| Ingress IPs do GoldenGate | Banco destino | TCP 1521 ou porta real | Apply / conexão Oracle |
| Deployment origem | Deployment destino | TCP 443 | Distribution Path remoto |

Para connections com **Shared endpoint**, use os *Ingress IPs* do Deployment. Para **Dedicated endpoint**, use os *Ingress IPs* exibidos nos detalhes da própria connection.

Se os bancos estiverem on-premises ou em outra VCN, valide DRG/LPG, VPN/FastConnect, DNS e rotas de retorno.

## 4. OCI Vault e secrets

### Criar o Vault

1. Acesse **Identity & Security → Vault → Create Vault**.
2. Informe um nome, por exemplo `VAULT-OGG`.
3. Aguarde o estado `Active`.

### Criar a chave

1. Abra o Vault.
2. Acesse **Master Encryption Keys → Create Key**.
3. Crie uma chave AES, por exemplo `KEY-OGG-SECRETS`.

### Criar os secrets

Crie secrets separados para cada finalidade:

```text
OGG-SRC-PASSWORD  → senha do GGADMIN na origem
OGG-TGT-PASSWORD  → senha do GGADMIN no destino
OGG-SRC-WALLET    → wallet da origem, se necessário
OGG-TGT-WALLET    → wallet do destino, se necessário
```

Não reutilize um secret de senha como wallet secret.

Se usar wallet, faça upload pelo botão **Create wallet secret** no fluxo de criação da connection. A wallet deve conter, no mínimo:

```text
cwallet.sso
tnsnames.ora
```

## 5. Preparar o Oracle Database origem

Execute com uma conta DBA e valide os comandos no processo de mudança da organização.

```sql
ARCHIVE LOG LIST;
```

O banco origem deve estar em `ARCHIVELOG`. Para captura GoldenGate, normalmente também são necessários:

```sql
ALTER DATABASE FORCE LOGGING;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
ALTER SYSTEM SET enable_goldengate_replication=TRUE SCOPE=BOTH;
```

Crie um usuário dedicado. Em uma base non-CDB, um exemplo inicial é:

```sql
CREATE USER ggadmin IDENTIFIED BY "<senha-segura>";
GRANT CREATE SESSION TO ggadmin;

BEGIN
  DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE(
    grantee                 => 'GGADMIN',
    privilege_type          => 'CAPTURE',
    grant_select_privileges => TRUE,
    do_grants               => TRUE);
END;
/
```

Em CDB/PDB, o DBA deve definir se o usuário será comum, por exemplo `C##GGADMIN`, e executar os comandos no container correto.

## 6. Preparar o Oracle Database destino

Crie o usuário dedicado no destino e conceda os privilégios de apply:

```sql
CREATE USER ggadmin IDENTIFIED BY "<senha-segura>";
GRANT CREATE SESSION TO ggadmin;

BEGIN
  DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE(
    grantee                 => 'GGADMIN',
    privilege_type          => 'APPLY',
    grant_select_privileges => TRUE,
    do_grants               => TRUE);
END;
/
```

Antes de iniciar o Replicat, confirme que schemas, tabelas, tipos de dados e chaves primárias/únicas sejam compatíveis com a origem.

## 7. Criar o Deployment

1. Acesse **Oracle AI Database → GoldenGate → Deployments → Create deployment**.
2. Selecione:

```text
Deployment type:  Data replication
Technology:       Oracle Database
Version:          compatível com origem e destino
Private subnet:   <subnet-golden-gate>
Credential store: OCI IAM ou GoldenGate
```

3. Aguarde o estado `Active`.
4. Registre a **Console URL**, os **Ingress IPs** e o endereço privado exibidos nos detalhes do Deployment.

Por padrão, o acesso à console é HTTPS na porta `443`. Acesso público deve ser habilitado apenas quando necessário e protegido por regras de rede restritivas.

## 8. Criar connections

Crie uma connection para a origem em **GoldenGate → Connections → Create connection**:

```text
Name:                       OGG-SRC-ORACLE
Type:                       Oracle Database
Connection string:          <host-ou-scan>:<porta>/<service-name>
Database username:          GGADMIN
Database password secret:   OGG-SRC-PASSWORD
Wallet secret:              OGG-SRC-WALLET, se TCPS/wallet
Network connectivity:       Shared endpoint ou Dedicated endpoint
```

Repita para o destino:

```text
Name:                       OGG-TGT-ORACLE
Type:                       Oracle Database
Connection string:          <host-ou-scan>:<porta>/<service-name>
Database username:          GGADMIN
Database password secret:   OGG-TGT-PASSWORD
Wallet secret:              OGG-TGT-WALLET, se TCPS/wallet
```

Crie e edite connections pela OCI Console. Evite editar credenciais diretamente na console interna do Deployment.

## 9. Atribuir e testar connections

1. Abra **GoldenGate → Deployments → `<deployment>` → Assigned connections**.
2. Clique em **Assign connection** e atribua a connection de origem e a de destino.
3. No menu de ações de cada connection, selecione **Test connection**.

O resultado deve confirmar:

```text
Network-level connectivity:      host e porta alcançáveis
Application-level connectivity:  credenciais e conexão Oracle válidas
```

## 10. Configurar a replicação

Abra **Launch console** no Deployment. A ordem geral é:

```text
Extract → Distribution Path → Replicat
```

### Origem

1. Crie um **Integrated Extract**.
2. Selecione a connection de origem.
3. Defina o ponto inicial: hora atual, SCN específico ou carga inicial planejada.
4. Habilite TRANDATA nos schemas/tabelas necessários.
5. Configure o trail local.
6. Crie o Distribution Path para o destino.

### Destino

1. Crie um **Replicat**.
2. Selecione a connection de destino.
3. Selecione o trail recebido.
4. Defina o mapeamento de schemas e tabelas.
5. Inicie o Replicat.

## 11. Carga inicial

Escolha uma estratégia antes de replicar alterações:

- Oracle Data Pump;
- Extract de carga inicial;
- ferramenta de carga externa;
- início em SCN definido, quando os dados já estiverem sincronizados.

Não inicie a replicação contínua sem definir como a carga inicial e o ponto de início do Extract serão coordenados.

## 12. Operação e validação

No Admin Client:

```text
INFO ALL
VIEW MESSAGES
```

Valide:

```text
Extract:   RUNNING
Replicat:  RUNNING
Lag:       aceitável para o SLA
Erros:     ausentes em VIEW MESSAGES
```

Execute um teste controlado de `INSERT`, `UPDATE` e `DELETE` na origem e confirme o resultado no destino.

## Troubleshooting rápido

| Sintoma | Causa provável | Ação |
|---|---|---|
| `Login timeout expired` | Listener, porta, NSG, firewall ou rota | Validar conectividade até a porta Oracle e os Ingress IPs |
| `Invalid walletSecretId` | Secret não contém wallet válida | Recriar o wallet secret com o arquivo correto |
| `unable to access secrets using resource principal` | Dynamic Group ou policy ausente | Revisar `read secret-bundles` |
| CIDR inválido | CIDR fora da VCN ou sobreposição | Escolher faixa livre dentro da VCN |
| Connection falha no teste de aplicação | Credencial, service name ou wallet incorretos | Validar connection string e secrets |

## Referências

- [OCI GoldenGate Policies](https://docs.oracle.com/en/cloud/paas/goldengate-service/ocigg/oracle-cloud-infrastructure-goldengate-policies.html)
- [OCI GoldenGate Connectivity](https://docs.oracle.com/en/cloud/paas/goldengate-service/ocigg/oci-goldengate-connectivity.html)
- [Connections para Oracle AI Database](https://docs.oracle.com/en/cloud/paas/goldengate-service/ocigg/connections/connect-to-oracle-ai-database.html)
