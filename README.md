# Helm Doc

Comandos essenciais do helm e criação de helm chart.

## Helm Repo

- Use o seguinte comando para procurar por um repositorio Helm. Outra alternativa é procurar pelo repositorio no site https://artifacthub.io/
``` bash
helm search hub <repo-name>
```

- Para adiciona um repositorio use o seguinte comando. O parametro <repo-name> é uma referencia ao <repo-url>, o nome pode ser adicionado de acordo com o usuario.
``` bash
helm repo add <repo-name> <repo-url>
```

- Para listar os charts de um repositorio use o seguinte comando. Alguns reporitorios contem diversos chart, como a bitnami, que contem os chart redis, postgresql, rabbitmq, etc.
``` bash
helm search repo <repo-name>
```

- Use o seguinte comando para lista os repositorios adicionados no seu computador.
``` bash
helm repo list
```

- Use o seguinte comando para remove um repositorio da maquina.
``` bash
helm repo remove <repo-name>
```

- Use o seguinte comando para atualiza os repositorios.
``` bash
helm repo update
```

## Helm Basic Commands

- Para fazer uma copia do arquivo de values default de um chart use o seguinte comando.
``` bash
helm show values <REPO/CHART-NAME> > values.yaml
```

- Para instala um helm chart use o seguinte comando.
  - release-name: nome de identificação da instalação do chart. Voce pode instalar o mesmo chart varias vezes, se for o caso crie uma boa identificação.
  - chart-name: Se estiver lidando com um chart em um reposotio siga o padrão "REPO/CHART" exemplo: bitnami/postgresql. Caso esteja utilizando um chart local que voce criou informe o caminho e o nome da pasta. OBS.: Não executar o comando de instalação e upgrade dentro da pasta do chart.
  - namespace: Namespace em que os serviços será criado.
- Se não for especificado um arquivo de values (-f values.yaml) o chart será instalado com o values default que esta dentro do chart.
``` bash
helm install <release-name> <chart-name> -n <namespace-name> --create-namespace

OR

helm install <release-name> <chart-name> -f values.yaml -n <namespace-name> --create-namespace 
```

- Para listar as releases dos helm charts já instalado, use o seguitne comando.
``` bash
helm list -A
```

- Para faz o upgrade de um helm chart ja instalado podemos usar duas opções.
  - values: Apos alterar o arquivo de values os valores default será modificado.
  - set: Caso não seja necessário um arquivo de value e apenas alguns valores será modificado pode ser usado o --set.
``` bash
helm upgrade <release-name> <chart-name> -f values.yaml -n <namespace-name>

OR

helm upgrade <release-name> <chart-name> --set image.name=nginx,image.tag=latest -n <namespace-name>
```

- A cada upgrade feito uma nova revisão da release é criada.
- Para fazer um rollback para a revisão anterior da release do chart use o seguinte comando.
- OBS.: Se voce estiver na revisão 3 ao fazer o rollback ele irá carregar as configurações da revisão 2 e criar a revisão 4.
``` bash
helm rollback <release-name> <revision> -n <namespace-name>
```

- Para saber as revisões da release de um chart.
``` bash
helm history <release-name> -n <namespace-name>
```

- Desinstala uma release.
``` bash
helm remove <release-name> -n <namespace-name>
```

## Helm Debug

- Para verificar se o helm chart e arquivo de values esta configurado corretamente use o seguinte comando. 
- Adicione a flag "--debug" para ter um output dos possiveis erros.
``` bash
helm template <chart-name>

OR

helm template <chart-name> --debug
```

- Verifica se o helm chart e values esta configurado corretamente usando um arquivo de values externo.
``` bash
helm template <chart-name> -f values.yaml

OR

helm template <chart-name> -f values.yaml --debug
```

## Create Helm Chart

- Use o seguinte comando para cria um novo chart. Obs: O chart será criado com templates e value de exemplo, remova os arquivos.
``` bash
helm create <chart-name> 
```

- Para definir variaveis em um template é usado a estrutura de linguagem GO.
  Note que as variáveis sempre se iniciam com ".Values" que é a referencia ao arquivo values.yaml.
  Caso algum dos valores não tiver a necessidade de ser configurado pode ser deixado como 'default' como no exemplo protocol e targetPort.
  Tambem é possivel definir uma variavel para o usuario definir o valor ou caso não seja definido será usado o valor default, como no exemplo 'port'.
``` yaml
# chart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.service.name }} # precisa ser especificado no values.yaml
  namespace: {{ .Values.namespace }} # precisa ser especificado no values.yaml
spec:
  selector:
    app: {{ .Values.deployment.name }} # precisa ser especificado no values.yaml
  ports:
    - protocol: TCP # valor default, não alteravel via values.yaml
      port: {{ default "80" .Values.service.port }} # Possue um valor default, mas pode ser alterado no values.yaml
      targetPort: 80 # valor default, não alteravel via values.yaml

# chart/values.yaml
namespace: nginx

deployment:
    name: nginx-deploy

service:
    name: nginx-service
    # port: 8080 # exemplo de alteração de uma variavel com valor default
```

- A condição "if" determina atravez de true ou false, se um trecho do código será adicionado ao template.
  No exemplo abaixo, se a variavel "replicas" for definida no arquivo de values, então ela será adicionada ao template.
  OBS.: note que a estrutura do "if" e "end"  se inicia com um sinal de "menos", isso evita a quebra de linha antes da linha do "if" e do "end". Se adicionado ao final, evita a quebra de linha depois.
``` yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  {{- if .Values.deployment.replicas }}
  replicas: {{ .Values.deployment.replicas }}
  {{- end }}
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

- A condição "with" determina se uma variavel (neste exemplo o resources) conter algum valor, caso possua o request e limits irá ser configurado, caso não tenha seu bloco é ignorado. 
``` yaml
# chart/templates/deployment.yaml
... # codigo omitido
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        {{- with .Values.deployment.resources }}
        resources:
          requests:
            cpu: {{ .Values.deployment.resources.reqcpu }}
            memory: {{ .Values.deployment.resources.reqmemory }}
          limits:
            cpu: {{ .Values.deployment.resources.limcpu }}
            memory: {{ .Values.deployment.resources.limmemory }}
        {{- end }}

# chart/values.yaml
deployment:
  # resources vazio não será adicionado ao deployment
  resources: {}

  # resources com parametros será adicionado ao deployment
  resources:
    reqcpu: 20m
    reqmemory: 100Mi

    limcpu: 50m
    limmemory: 200Mi
```

- A condição "range" possibilita repetir um determinado trecho do arquivo dinamicamente. No exemplo abaixo é definido apenas uma vez 'name' e 'port', e no arquivo de values podemos definir quantas portas queromos expor.
``` yaml
# chart/templates/service.yaml
... # codigo omitido
  ports:
    - protocol: TCP
      {{- range .Values.service.ports }}
      name: {{ .name }}
      port: {{ .port }}
      {{- end }}

# chart/values.yaml
service:
  ports:
    - name: 80
      port: 80
    - name: 443
      port: 443
```

- Para criar uma variavel local em um template inicie com o simbolo '$'. Em alguns momentos é necessário quando utlizado o 'with' pois voce não consegue acessar variaveis fora do que for definido no 'with'.
``` yaml
# chart/templates/deployment.yaml
{{- $namespace := .Values.OverrideNamespace -}}

{{- with .Values.configMap }}
{{- if .enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .name }}
  namespace: {{ default $.Release.Namespace $namespace }}
data:
  {{- range $key, $value := .data }}
    {{ $key }}: {{ $value }}
  {{- end }}
{{- end }}
{{- end }}
```

## Chart Version

- Para criar um package do seu helm chart use o seguinte comando. Irá gerar um arquivo my-chart-1.0.0.tgz. Obs: a versão "1.0.0" será configurada de acordo com a versão definida no arquivo Chart.yaml
``` bash
helm package <chart-name>
```

- Para conseguir o digest do helm chart use o seguinte comando. Este digest é alterado a cada modificação do chart.
``` bash
sha256sum my-chart-1.0.0.tgz
```

- Use o seguinte comando para gerar um timestamp para ser usado a seguir.
``` bash
echo "$(TZ=America/Sao_Paulo date +"%Y-%m-%dT%H:%M:%S.%N%:z" | sed 's/\(...\)$/:00/')"
```

- Crie um arquivo 'index.yml' na raiz do projeto junto com o helm chart e o package. Ele é responsavel por determinar as versões do chart possiveis de ser usado.
- Atualize os campos a cada nova versão criada do chart:
``` yaml
apiVersion: v1
entries:
  <CHART NAME>: # Nome do seu chart
    ## Version 1
  - apiVersion: v2
    appVersion: "1.0" # Versão definida dentro do arquivo Chart.yaml
    created: "<TIMESTAMP>"
    description: Helm chart to deployment applications on Kubernetes
    digest: <DIGUEST> # Digest coletado com o comando anterior
    home: https://github.com/<REPO NAME> # url do repositorio
    name: <CHART NAME> # Nome do seu chart
    sources:
    - https://github.com/<REPO NAME> # url do repositorio
    urls:
    - <CHART.TAR.GZ> # Nome do package gerado anteriormente
    version: <CHART VERSION> # Versão definida dentro do arquivo Chart.yaml
    ## Version 2
  - apiVersion: v2
    appVersion: "1.0" # Versão definida dentro do arquivo Chart.yaml
    created: "<TIMESTAMP>"
    description: Helm chart to deployment applications on Kubernetes
    digest: <DIGUEST> # Digest coletado com o comando anterior
    home: https://github.com/<REPO NAME> # url do repositorio
    name: <CHART NAME> # Nome do seu chart
    sources:
    - https://github.com/<REPO NAME> # url do repositorio
    urls:
    - <CHART.TAR.GZ> # Nome do package gerado anteriormente
    version: <CHART VERSION> # Versão definida dentro do arquivo Chart.yaml
```