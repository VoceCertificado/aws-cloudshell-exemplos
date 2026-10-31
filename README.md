[![AWS CloudShell](https://img.shields.io/badge/AWS-CloudShell-FF9900?style=flat-square)](https://docs.aws.amazon.com/cloudshell/latest/userguide/welcome.html)
[![AWS CLI](https://img.shields.io/badge/AWS-CLI-232F3E?style=flat-square)](https://docs.aws.amazon.com/cli/latest/reference/)
[![AWS CloudFormation](https://img.shields.io/badge/AWS-CloudFormation-FF9900?style=flat-square)](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)
[![Bash](https://img.shields.io/badge/Shell-Bash-4EAA25?style=flat-square)](https://docs.aws.amazon.com/cloudshell/latest/userguide/vm-specs.html)

# AWS CloudShell: 10 comandos práticos

Aprenda dez comandos práticos para usar no AWS CloudShell e conhecer melhor seu ambiente AWS.

Com eles, você gerará um inventário dos Amazon EC2 da região conectada, encontrar volumes Amazon EBS desconectados, investigar quem interrompeu as instâncias e consultar informações de segurança, monitoramento e armazenamento do Amazon S3.

Faça o download e utilizar o template [lab-cloudshell.yaml](./lab-cloudshell.yaml), para preparar um ambiente temporário e controlado com o AWS CloudFormation e que deve ser iniciado na região **Norte da Virgínia (`us-east-1`)**. 

## Custos e exclusão do ambiente

> [!WARNING]
> **O ambiente criado pelo template gera custos. Ao terminar os testes, exclua a pilha pelo AWS CloudFormation.** Parar a EC2 não remove os volumes EBS, e o Elastic IP continua alocado até ser liberado. Não considere este laboratório gratuito.

Podem existir cobranças pelo uso da EC2, pelos volumes EBS, pelo Elastic IP, pelo alarme do CloudWatch e pelos serviços de automação. O uso do AWS CloudShell também poderá gerar cobranças por requisições de listagem.

Porém, os custos gerados são mínimos, e em testes controlados gerou centavos de dólar.

> [!NOTE]
> Se você não excluir o ambiente manualmente, o template está configurado para **fazer a exclusão automática da pilha 10 minutos após o Amazon EC2 ficar parado**. O horário programado aparece em **Saídas → `DeletionTimeUtc`**, em UTC. O agendamento tem precisão de um minuto, e a exclusão dos recursos leva um tempo adicional.

A exclusão automática depende da criação bem-sucedida do agendamento e das permissões necessárias. Se a implantação falhar antes dessa etapa, a exclusão automática pode não ser configurada. Por isso, confira o estado final da pilha no AWS CloudFormation e siga a seção [Excluir o laboratório](#excluir-o-laboratório).

## Antes de começar

- Tenha uma conta AWS e acesso ao AWS CloudShell.
- Tenha uma credencial com permissão administrativa: `AdministratorAccess`.
- Selecione a região **`us-east-1`**, Norte da Virgínia.
- Use o shell **Bash**, com AWS CLI e `jq`, disponíveis no ambiente padrão do CloudShell.

As consultas regionais mostram recursos da **região atual** e podem incluir outros recursos da sua conta. 

Por exemplo, o comando do Amazon S3 listará buckets de uso geral da conta e consultará os objetos dos buckets aos quais a sua identidade possui acesso, inclusive de outras regiões.

Resultados vazios podem indicar ausência de recursos ou de eventos correspondentes. E erros `AccessDenied` indicam falta de permissão.

## Criar o laboratório

1. Baixe o arquivo [lab-cloudshell.yaml](./lab-cloudshell.yaml).
2. No console AWS selecione a região **Norte da Virgínia (`us-east-1`)**.
3. Abra o **AWS CloudFormation** e clique no botão **Criar pilha → Fazer upload de um arquivo de modelo** e faça o upload do arquivo YAML.
4. Na **Etapa 2**, em nome da pilha digite **`cloudshell`** e pressione o botão próximo.
5. Na **Etapa 3**, desça a barra de rolagem e marque a caixa de seleção `Entendo que o AWS CloudFormation pode criar recursos do IAM com nomes personalizados`, seguido do botão `Próximo`.
6. Na **Etapa 4**, desça a barra de rolagem e pressione o botão `Enviar`.
7. Aguarde o estado **`CREATE_COMPLETE`**. Confira o horário em **Saídas → `DeletionTimeUtc`** e execute os comandos abaixo, antes da exclusão programada de 10 minutos.

  ![Eventos da pilha no AWS CloudFormation](/imagens/eventos-cloudformation.webp)

### Recursos preparados

| Recurso | Finalidade |
| --- | --- |
| VPC, subnet e security group | Isolar o laboratório e demonstrar uma regra de entrada TCP/80 com origem `0.0.0.0/0` |
| EC2 `t2.micro` com Amazon Linux 2023 | Aparecer no inventário e gerar uma chamada `StopInstances` ao ser parada automaticamente |
| Volume raiz EBS de 8 GiB | Armazenamento da EC2, removido quando a instância é terminada |
| Volume EBS `gp3` de 1 GiB, desconectado | Aparecer na consulta de volumes no estado `available` |
| Elastic IP sem associação | Aparecer na consulta de endereços sem associação |
| Função Lambda demonstrativa e função de preparação | Demonstrar a consulta de runtimes e executar a preparação automática |
| Alarme do CloudWatch | Entrar em `ALARM` tratando a ausência de dados como violação, sem executar ações |
| Grupos de logs e roles IAM | Dar suporte às funções e à exclusão do laboratório |
| Agendamento único e grupo do EventBridge Scheduler | Solicitar a exclusão da pilha no horário calculado |

O Amazon EC2 não recebe IP público, e a VPC não possui Internet Gateway nem NAT Gateway. A regra de entrada é apenas demonstrativa. O alarme também é demonstrativo e não representa uma falha real da aplicação.

O template **não cria buckets nem arquivos no S3**. E a consulta de cotas usa informações existentes da sua conta, sem alterar os limites.

## Os 10 exemplos práticos

### 1. Exportar um inventário EC2 para CSV

Lista ID, tipo, estado e IP privado das instâncias da região e gera o arquivo `inventario-ec2.csv` no diretório atual. O `jq` organiza a resposta em CSV, incluindo o cabeçalho. Se o arquivo já existir, será sobrescrito.

```bash
aws ec2 describe-instances --output json |
jq -r '
  ["ID","Tipo","Estado","IP privado"],
  (.Reservations[].Instances[] |
    [.InstanceId,.InstanceType,.State.Name,.PrivateIpAddress])
  | @csv
' > inventario-ec2.csv
```

O comando executado no AWS CloudShell procurará por instâncias EC2 na região conectada, salvará o resultado no arquivo ``inventario-ec2.csv`` e a sua leitura é realizado com:

```bash
cat inventario-ec2.csv
```


### 2. Encontrar volumes EBS desconectados

Lista os volumes no estado `available`, mostrando o identificador, o tamanho em GiB e o seu tipo.

```bash
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[].{ID:VolumeId,TamanhoGiB:Size,Tipo:VolumeType}' \
  --output table
```

O volume adicional de 1 GiB criado neste laboratório aparecerá nessa consulta.

![Exemplo do comando](/imagens/cli-exemplo-ebs.webp)

### 3. Investigar solicitações de parada de instâncias EC2

Consulta os dez eventos `StopInstances` mais recentes disponíveis no histórico regional do CloudTrail, exibindo data, identidade e recursos associados. O histórico abrange os últimos 90 dias.

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=StopInstances \
  --max-items 10 \
  --query 'Events[].{Quando:EventTime,Identidade:Username,Recursos:Resources}' \
  --output json
```

Neste laboratório, a parada é solicitada por uma função, portanto a identidade registrada que será apresentada será `nome-da-pilha-lab`. O evento pode levar alguns minutos para aparecer e continua no histórico depois da exclusão da pilha.

![Exemplo do comando](/imagens/cli-exemplo-ec2.webp)

### 4. Localizar regras de entrada abertas para qualquer IP

Mostra regras de entrada de security groups com origem IPv4 `0.0.0.0/0` ou IPv6 `::/0`, incluindo protocolo e intervalo de portas.

```bash
aws ec2 describe-security-group-rules \
  --query 'SecurityGroupRules[?IsEgress==`false` && (CidrIpv4==`0.0.0.0/0` || CidrIpv6==`::/0`)].{Grupo:GroupId,Protocolo:IpProtocol,De:FromPort,Ate:ToPort,IPv4:CidrIpv4,IPv6:CidrIpv6}' \
  --output table
```

A regra TCP/80 do laboratório deverá aparecer. A existência de uma regra aberta, isoladamente, não comprova que um recurso esteja acessível pela internet; também é necessário considerar rotas, endereços e outros controles de rede.

![Exemplo do comando](/imagens/cli-exemplo-sg.webp)

### 5. Encontrar Elastic IPs sem associação

Lista endereços Elastic IP alocados na conta, na região atual, que não possuem associação. Esses endereços devem ser revisados, pois continuam gerando cobrança até serem liberados.

```bash
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==`null`].{IP:PublicIp,Alocacao:AllocationId}' \
  --output table
```

O Elastic IP criado pelo laboratório permanece sem associação para aparecer nesse resultado.

![Exemplo do comando](/imagens/cli-exemplo-eip.webp)

### 6. Identificar alarmes de métricas em alerta

Exibe os alarmes de métricas no estado `ALARM`, com o motivo informado pelo CloudWatch.

```bash
aws cloudwatch describe-alarms \
  --state-value ALARM \
  --query 'MetricAlarms[].{Alarme:AlarmName,Motivo:StateReason}' \
  --output table
```

O alarme do laboratório usa `TreatMissingData: breaching` e deve entrar em `ALARM` após a avaliação. Ele não envia notificações nem executa ações sobre a EC2. Este exemplo apresenta alarmes de métricas, sem listar alarmes compostos.

![Exemplo do comando](/imagens/cli-exemplo-alarme.webp)

### 7. Consultar cotas ajustáveis do Amazon EC2

Lista somente as cotas ajustáveis do EC2 com valor maior que zero. Os valores representam limites disponibilizados pelo Service Quotas, e não o consumo atual de cada recurso.

```bash
aws service-quotas list-service-quotas \
  --service-code ec2 \
  --query 'Quotas[?Adjustable==`true` && Value>`0`].{Cota:QuotaName,Valor:Value,Ajustavel:Adjustable}' \
  --output table --no-cli-pager
```

![Exemplo do comando](/imagens/cli-exemplo-cota.webp)

### 8. Encontrar os dez maiores objetos de cada bucket S3

Descobre automaticamente os buckets de uso geral da conta e lista até dez objetos de maior tamanho em cada bucket. O script preenche suas variáveis internamente, sem exigir que você informe nomes. Se uma consulta falhar, informa o problema e continua para o próximo bucket.

```bash
(
  set -euo pipefail

  buckets=$(
    aws s3api list-buckets \
      --output json \
      --no-cli-pager
  )

  if [ "$(jq '.Buckets | length' <<< "$buckets")" -eq 0 ]; then
    printf 'Nenhum bucket encontrado.\n'
    exit 0
  fi

  while IFS= read -r bucket; do
    printf '\nBucket: %s\n' "$bucket"

    if objetos=$(
      aws s3api list-objects-v2 \
        --bucket "$bucket" \
        --output json \
        --no-cli-pager
    ); then
      jq '
        (.Contents // []) |
        sort_by(.Size) |
        reverse |
        .[:10] |
        map({
          Arquivo: .Key,
          TamanhoMiB: ((.Size / 1048576 * 100 | round) / 100)
        })
      ' <<< "$objetos"
    else
      printf 'Não foi possível consultar este bucket.\n'
    fi
  done < <(jq -r '.Buckets[].Name' <<< "$buckets")
)
```

O resultado mostra os tamanhos em **MiB** e considera os objetos atuais, sem versões anteriores nem uploads multipart incompletos. Buckets vazios retornam `[]`. O script consulta metadados, sem baixar os arquivos.

**Atenção ao volume de dados:** para encontrar os maiores objetos, o script lista todos os objetos atuais de cada bucket antes de ordenar. Em buckets grandes, isso pode demorar, consumir memória no CloudShell e gerar muitas requisições de listagem. O script não contempla S3 directory buckets.

![Exemplo do comando](/imagens/cli-exemplo-s3.webp)

### 9. Conferir os runtimes das funções Lambda

Lista nome, runtime e memória configurada das funções Lambda da região.

```bash
aws lambda list-functions \
  --query 'Functions[].{Funcao:FunctionName,Runtime:Runtime,MemoriaMB:MemorySize}' \
  --output table
```

A listagem apresentará a função demonstrativa e a função de laboratório, ambas com Node.js 22 e 128 MB de memória. Funções implantadas como imagens de contêiner podem não preencher o campo `Runtime`.

![Exemplo do comando](/imagens/cli-exemplo-lambda.webp)

### 10. Verificar o estado das stacks do CloudFormation

Exibe nomes e estados das pilhas existentes na região atual. Durante os testes, a pilha do laboratório deve aparecer como `CREATE_COMPLETE`; após o início da limpeza, pode aparecer como `DELETE_IN_PROGRESS`.

```bash
aws cloudformation describe-stacks \
  --query 'Stacks[].{Stack:StackName,Estado:StackStatus}' \
  --output table
```

Depois que a exclusão termina, a pilha deixa de aparecer nessa consulta.

![Exemplo do comando](/imagens/cli-exemplo-pilha-cloudformation.webp)

## Excluir o laboratório

**Ao concluir os comandos no AWS CloudShell, exclua a pilha pelo AWS CloudFormation.**.

### Pelo console de gerenciamento AWS

1. Selecione a região `us-east-1` (Norte da Virgínia)
2. Abra o **AWS CloudFormation**.
3. Selecione a pilha **`cloudshell`**.
4. Escolha **Excluir** e confirme.
5. Aguarde o estado **`DELETE_COMPLETE`**.

### Pelo AWS CloudShell

1. Solicite a exclusão com o comando:

```bash
aws cloudformation delete-stack \
  --stack-name cloudshell \
  --region us-east-1
```

Acompanhe a conclusão:

```bash
aws cloudformation wait stack-delete-complete \
  --stack-name cloudshell \
  --region us-east-1
```

O comando de espera não exibe progresso e retorna sem erro quando a exclusão termina. Se houver falha, consulte os eventos da pilha no console e corrija a causa antes de concluir a limpeza.

`Nota:` Não remova nenhum recurso manualmente. Faça a limpeza desse laboratório através da exclusão da pilha via AWS CloudFormation ou via AWS CloudShell.

### Remover a role IAM retida

A role **`cloudshell-exclusao-cfn`** usa `DeletionPolicy: Retain` para preservar as permissões necessárias enquanto a exclusão ocorre. Ela e sua policy inline permanecem após a exclusão da pilha e devem ser removidas manualmente no final.

**Execute o bloco abaixo somente após confirmar que a pilha foi excluída.**

```bash
aws cloudformation wait stack-delete-complete \
  --stack-name cloudshell \
  --region us-east-1 &&
aws iam delete-role-policy \
  --role-name cloudshell-exclusao-cfn \
  --policy-name CleanupLab &&
aws iam delete-role \
  --role-name cloudshell-exclusao-cfn
```

`Nota:` Remova essa função antes de recriar o laboratório com o mesmo nome de pilha, para evitar conflito de nome.

## Erros comuns e soluções

- **Falta de capacidade para `t2.micro`:** escolha outra zona no parâmetro `LabAvailabilityZone` e tente novamente. A disponibilidade varia ao longo do tempo.
- **Pilha em `ROLLBACK_COMPLETE`:** exclua a pilha com falha antes de criar uma nova. Confira se a role retida foi criada e remova-a, se necessário, depois da exclusão.
- **Erro de permissão:** consulte a mensagem nos eventos da pilha e verifique as permissões da identidade utilizada. O template cria roles para a automação, mas não concede permissões adicionais à identidade conectada ao CloudShell.
- **Exclusão automática não iniciada:** confira `DeletionTimeUtc`, o estado do agendamento no EventBridge Scheduler e a proteção contra exclusão da pilha. Faça a exclusão manual pelo CloudFormation se necessário.
- **`AccessDenied` ao excluir o grupo do Scheduler:** a role de exclusão precisa de `scheduler:DeleteSchedule` sobre `schedule/<nome-do-grupo>/*`, além de `scheduler:DeleteScheduleGroup` sobre o grupo. A exclusão do grupo abrange seus agendamentos. O template deste repositório inclui essa permissão, restrita ao grupo do laboratório. Se você criou a pilha com uma versão anterior, corrija o recurso da declaração `ExcluirAgendamentoDoLaboratorio` na policy inline `CleanupLab` da role de exclusão e tente excluir a pilha novamente pelo CloudFormation.

## Documentação oficial

- [AWS CloudShell](https://docs.aws.amazon.com/cloudshell/latest/userguide/welcome.html)
- [Referência de comandos da AWS CLI](https://docs.aws.amazon.com/cli/latest/reference/)
- [Histórico de eventos do CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/view-cloudtrail-events.html)
- [Agendamentos do EventBridge Scheduler](https://docs.aws.amazon.com/scheduler/latest/UserGuide/schedule-types.html)
- [Excluir uma pilha CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html)
- [Elastic IPs e cobranças](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)

## Autoria

Desenvolvido por **Eduardo Lara**, da [Você Certificado](https://www.vocecertificado.com.br/), para fins educacionais.
