# Configuração Integração Contínua + Execução de Testes Automatizados via Gitlab-CI

No servidor que será executado o processo instalar:

  - [NuGet] - responsável por baixar os pacotes 
  - [MSBuild] - responsável por realizar a construção do projeto de testes
  - [NUnit3-console] - responsável por executar os testes do projeto de testes
  - [Gitlab-Runner] - responsável por gerenciar/executar  o processo de integração contínua
 
### 1. Processo de integração contínua e testes automatizados

O processo de integração contínua e testes automatizados é feita com os seguintes passos:
  1. Baixar os pacotes do projeto de testes
  2. Construir o projeto de testes
  3. Configurar o relatório de execução dos testes ([ExtentReport]) para a base que será testada
  4. Remover a dll.config padrão do projeto de testes
  5. Renomear a dll.config da base que será testada
  6. Mover a dll.config da base que será testada para o diretório correto
  7. Executar os testes automatizados na base que será testada
  8. Enviar um email com as informações da execução dos testes automatizados

Estes passos serão configurados através de um arquivo de configuração chamado de "gitlab-ci.yml", o mesmo deverá ser gerado dentro do repositório do projeto de testes automatizados no gitlab.

### 2. Configuração arquivo gitlab-ci.yml

Primeiramente, no repositório de testes automatizados, crie um novo arquivo:
[![N|Solid](https://i.imgur.com/MXBgAPC.png)]()
Escolha o tipo de arquivo "gitlab-ci.yml":
[![N|Solid](https://i.imgur.com/v9Dh3L9.png)]()

O arquivo de configuração deverá ter as informações discriminadas por JOBs onde, cada JOB depende do outro como pré-requisito para execução do próximo. Os passos listados no primeiro tópico deverão ser dispostos no arquivo como forma de organizar a execução passo a passo.

Teremos dois estágios para o arquivo de configuração:
 - **build**: passos referentes à construção do projeto de testes automatizados
 - **test**: passos referentes à execução dos testes automatizados
 
**Para isso devemos iniciar nosso arquivo com o seguinte trecho de código:**
```
stages:
  - build
  - test
```


##### 2.1 Organização dos jobs

Após definir quais estágios serão executados pelo arquivo de yml, devemos definir quais jobs serão executados.
No caso, o estágio de **build** deverá ser feito somente uma vez independente da quantidade de ambientes que os testes serão executados, pois o que vai diferenciar a execução de um ambiente para o outro é a conexão com o banco de dados para busca de informações e link de acesso disponibilizado pelo projeto de desenvolvimento (independente).

**A organização dos jobs para que seja executado testes em três diferentes ambientes ficará da seguinte forma:**
[![N|Solid](https://i.imgur.com/GCy9UP1.png)]()

Dois estágios com 4 jobs para serem executados:
- Build: 
  - projeto_teste
- Test:
  - Ambiente acadêmico
  - Ambiente financeiro
  - Ambiente suporte

##### 2.2 Organização dos jobs: build
Abaixo o código representando todos os passos necessários para realizar a build do projeto de testes automatizados. 
 - No exemplo é possível verificar o uso da tag **only**, esta representa quando será executado os testes automatizados, o uso de tags representa quando você quer executar o processo:
   - **schedules:** representa a execução somente programada dentro do Gitlab (tela Pipeline - Schedules).
   - **web:** representa a execução somente quando o botão "Run Pipeline"  for executado presente na tela de Pipelines do Gitlab.
  
- Tag **stage: build** faz referência em qual estágio o job será executado.
- Tag **tags** define em qual ambiente será executado o job, podemos ter vários ambientes definidos por tags. No exemplo temos um servidor (windows) que executa a build do projeto de testes automatizados. Posteriormente veremos que para o estágio **test** será usado outro servidor para executar os testes automatizados, a justificativa é por conta da alocação do runner durante a execução dos testes, o que pode gerar um gargalo pelo tempo de execução (superior à 15 minutos por execução).
- Tag **script** resume o que será executado no job, para o job **build:projeto_teste** executaremos os seguintes passos previamente definidos na sessão 1:
  1. Baixar os pacotes do projeto de testes: NuGet
  2. Construir o projeto de testes: MSBuild
- Tag **artifacts** cria um pacote com tudo que foi criado para ser usado em outros ambientes (servidores)
  - A subtag **paths** indica o diretório que deverá ser "empacotado" para ser usado nos demais ambientes.
  
```
build:projeto_teste:
  only: 
   - schedules
   - web
  stage: build
  tags:
   - windows
  script:
  #Restore Nuget
   - '"C:\\Gitlab-Runner\\nuget.exe" restore "test-automation-inscricao-vestib.sln"'

  #Build do projeto no ambiente WINDOWS (servidor de build)
   - '"C:\\Program Files (x86)\\MSBuild\\14.0\\Bin\\msbuild.exe" /t:Clean,Build /p:Configuration=Debug "test-automation-inscricao-vestib.sln"'
  artifacts:
   paths:
    - test-automation-inscricao-vestib\bin\Debug

```


##### 2.1 Organização dos jobs: test
Abaixo o código representando todos os passos necessários para realizar a execução dos testes automatizados. A organização foi feita em tags e feita em etapas até ser feita a execução da bateria de testes.

- No exemplo é possível ver que foi criado um stage do tipo **test**, este tipo foi definido no ínicio do arquivo **yml** simbolizando a execução após do estágio **build**
- Tag **only** está sendo usado dois parâmetros:
    -  schedules: execução do job somente quando for programado.
    -  web: execução do job pelo site gitlab sempre que necessário
- Tag **tags** neste job recebeu o valor **teste**, este valor foi configurado para que o SERVIDOR DE EXECUÇÃO DOS TESTES pudesse ser utilizado após o build do projeto de testes automatizados ser feito. A escolha de usar um outro servidor para executar foi mediante gargalo que poderia causar no servidor de build (**windows**).
- Script de teste: o script de execução dos testes automatizados segue os seguintes passos:
    - Limpeza caso exista, do relatório gerado via NUnit criados no servidor (**TestResult.xml**).
    - Limpeza caso exista, do relatório de testes automatizados (framework ExtentReport) no servidor.
    - Remoção do arquivo de configuração (**App.config**) gerado pelo build (stage anterior). Isto é feito para ser utilizado um arquivo de configuração correto conforme o AMBIENTE QUE SERÁ EXECUTADO.
    - Renomear o arquivo de configuração do ambiente que será utilizado: renomeado para o padrão do projeto, neste caso "Nome-do-projeto.dll.config".
    - Mover o arquivo de configuração remoneado para a pasta que ele vai ser lido/usado. Passo importante para que seja feito o uso da configuração correta do ambiente.
    - Acessar a pasta **Debug** do projeto.
    - Executar os testes automatizados via NUnit3-console com uma chamada explicita da dll do projeto. Para esta linha usou-se os seguintes parâmetros:
    - **"C:\\Program Files (x86)\NUnit.org\nunit-console\nunit3-console.exe"**: usa-se o diretório do NUnit3-console para executá-lo.
    - **"test-automation-inscricao-vestib.dll"**: a DLL que será executada, ou seja, aqui o sistema entende quais testes deverão ser executados.
    - **--inprocess**: parâmetro que se refere ás configurações atuais do processo. Utilizado para ler a configuração correta do App.config.
    - **--labels=On**: á definir
    - **--where "cat==Producao"**: parâmetro para indicar qual categoria de teste será executada. No meu código fonte, os métodos de testes tem a decoração indicando este comportamento. Exemplo: [Category("**Producao**"), TestCaseSource("ConfiguracaoInstituicoes")].
    - **--work:"C:\LogsAutomacao\InscricaoVestib"**: pârametro que indica onde será salvo o arquivo de execução dos testes automatizados (**TestResult.xml**).

Feito tal execução dos testes. Foi arquitetado o envio de relatório de execução utilizando um arquivo via powershell. O mesmo está descrito na tag **after_script**.
- Acessa o diretório do software de envio
- Envia o arquivo passando determinados parâmetros:
    - **.\SendReport_versao2.ps1**: nome do arquivo
    - **-branch "%CI_BUILD_REF_NAME%"** branch do projeto recebido de forma dinâmica
    - **-ambiente Academico**: ambiente que está sendo executado.

No final é descrito uma dependência, isto é, tal job só é executado se o job anterior (build) for feito com sucesso.

```
test:academico:
  only: 
   - schedules
   - web
  stage: test
  tags:
   - teste
  script:
   - powershell if (Test-Path C:\LogsAutomacao\InscricaoVestib\TestResult.xml) { rm C:\LogsAutomacao\TestResult.xml }
   - powershell if (Test-Path C:\LogsAutomacao\InscricaoVestib\Academico\Report_Vestib_Academico.html) { rm C:\LogsAutomacao\InscricaoVestib\Academico\Report_Vestib_Academico.html }
   - powershell Remove-Item test-automation-inscricao-vestib\bin\Debug\test-automation-inscricao-vestib.dll.config
   - powershell Rename-Item test-automation-inscricao-vestib\test-automation-inscricao-vestib_Academico.dll.config test-automation-inscricao-vestib.dll.config
   - powershell Move-Item test-automation-inscricao-vestib\test-automation-inscricao-vestib.dll.config test-automation-inscricao-vestib\bin\Debug
   
   - cd test-automation-inscricao-vestib/bin/Debug
  - '"C:\\Program Files (x86)\NUnit.org\nunit-console\nunit3-console.exe"   - '"C:\\Program Files (x86)\NUnit.org\nunit-console\nunit3-console.exe" "test-automation-inscricao-vestib.dll" --inprocess --labels=On --where "cat==Producao" --work:"C:\LogsAutomacao\InscricaoVestib"'

  after_script:
   - cd C:\SendReport
   - powershell -noexit ".\SendReport_versao2.ps1 -branch "%CI_BUILD_REF_NAME%" -ambiente Academico"
  dependencies:
   - build:test   
```


### 3. Arquivo gitlab-ci.yml completo
Segue abaixo o arquivo completo gerado para este processo de execução de testes automatizados via gitlab + ci.
```
stages:
  - build
  - test

build:test:
  only: 
   - schedules
   - web
  stage: build
  tags:
   - windows
  script:
  #Restore Nuget
   - '"C:\\Gitlab-Runner\\nuget.exe" restore "test-automation-inscricao-vestib.sln"'

  #Build do projeto no ambiente WINDOWS (servidor de build)
   - '"C:\\Program Files (x86)\\MSBuild\\14.0\\Bin\\msbuild.exe" /t:Clean,Build /p:Configuration=Debug "test-automation-inscricao-vestib.sln"'
  artifacts:
   paths:
    - test-automation-inscricao-vestib\bin\Debug
  
test:academico:
  only: 
   - schedules
   - web
  stage: test
  tags:
   - teste
  script:
   - powershell if (Test-Path C:\LogsAutomacao\InscricaoVestib\TestResult.xml) { rm C:\LogsAutomacao\TestResult.xml }
   - powershell if (Test-Path C:\LogsAutomacao\InscricaoVestib\Academico\Report_Vestib_Academico.html) { rm C:\LogsAutomacao\InscricaoVestib\Academico\Report_Vestib_Academico.html }
   - powershell Remove-Item test-automation-inscricao-vestib\bin\Debug\test-automation-inscricao-vestib.dll.config
   - powershell Rename-Item test-automation-inscricao-vestib\test-automation-inscricao-vestib_Academico.dll.config test-automation-inscricao-vestib.dll.config
   - powershell Move-Item test-automation-inscricao-vestib\test-automation-inscricao-vestib.dll.config test-automation-inscricao-vestib\bin\Debug
   
   - cd test-automation-inscricao-vestib/bin/Debug
   - '"C:\\Program Files (x86)\NUnit.org\nunit-console\nunit3-console.exe" "test-automation-inscricao-vestib.dll" --inprocess --labels=On --where "cat==Producao" --work:"C:\LogsAutomacao\InscricaoVestib"'

  after_script:
   - cd C:\SendReport
   - powershell -noexit ".\SendReport_versao2.ps1 -branch "%CI_BUILD_REF_NAME%" -ambiente Academico"
  dependencies:
   - build:test   
```








   [NuGet]: <https://www.nuget.org/downloads>
   [MSBuild]: <https://www.microsoft.com/en-us/download/details.aspx?id=48159>
   [NUnit3-console]: <http://nunit.org/download/>
   [Gitlab-Runner]: <https://docs.gitlab.com/runner/install/windows.html>
   [ExtentReport]: <http://extentreports.com/docs/versions/2/net/>
   [markdown-it]: <https://github.com/markdown-it/markdown-it>
   [Ace Editor]: <http://ace.ajax.org>
   [node.js]: <http://nodejs.org>
   [Twitter Bootstrap]: <http://twitter.github.com/bootstrap/>
   [jQuery]: <http://jquery.com>
   [@tjholowaychuk]: <http://twitter.com/tjholowaychuk>
   [express]: <http://expressjs.com>
   [AngularJS]: <http://angularjs.org>
   [Gulp]: <http://gulpjs.com>

   [PlDb]: <https://github.com/joemccann/dillinger/tree/master/plugins/dropbox/README.md>
   [PlGh]: <https://github.com/joemccann/dillinger/tree/master/plugins/github/README.md>
   [PlGd]: <https://github.com/joemccann/dillinger/tree/master/plugins/googledrive/README.md>
   [PlOd]: <https://github.com/joemccann/dillinger/tree/master/plugins/onedrive/README.md>
   [PlMe]: <https://github.com/joemccann/dillinger/tree/master/plugins/medium/README.md>
   [PlGa]: <https://github.com/RahulHP/dillinger/blob/master/plugins/googleanalytics/README.md>
