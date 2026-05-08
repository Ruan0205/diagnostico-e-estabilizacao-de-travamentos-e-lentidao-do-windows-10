# Correção de falha da Microsoft Store e do Windows Update no Windows 10

## Resumo

Durante uma tentativa de instalar o WhatsApp para Windows pela Microsoft Store, foi identificado um problema estrutural no mecanismo de instalação de aplicativos da Store. O WhatsApp não era a causa do incidente; ele apenas expôs uma falha pré-existente na integração entre Microsoft Store, Windows Update, serviços DCAT e validação de certificados do Windows.

O problema impedia que pacotes da Microsoft Store fossem adquiridos e instalados corretamente. A falha principal aparecia como `0x80248014` ao instalar pela Store ou pelo `winget`, mas a investigação mostrou que esse erro era consequência de problemas anteriores na pilha de atualização e confiança criptográfica do sistema.

## Ambiente

- Sistema operacional: Windows 10 Pro
- Build: `19045`
- Arquitetura: `x64`
- Ferramentas envolvidas: Microsoft Store, Windows Update, App Installer, `winget`, CryptoAPI, `CryptSvc`, `InstallService`
- Aplicativo usado como gatilho: WhatsApp para Windows (`9NKSQGP7F2NH`)

## Sintomas observados

A instalação do WhatsApp pela Microsoft Store falhava com:

```text
Falha ao instalar ou atualizar o pacote da Microsoft Store.
Código de erro: 0x80248014
```

Ao investigar a Store e o Windows Update, também foram observados:

- `0x80072F8F` em buscas diretas pelo Windows Update.
- `0x80092013` em validações TLS/Schannel, indicando falha na verificação de revogação de certificados.
- Fonte da Microsoft Store/DCAT ausente no banco do Windows Update.
- Fonte `winget` travando ao usar IPv6 para endpoints da Microsoft.
- Item de instalação do WhatsApp preso na fila interna da Store em estado `Error`.

## Diagnóstico

O erro `0x80248014` indicava que o Windows Update não conseguia resolver o serviço federado usado pela Microsoft Store, especialmente a fonte:

```text
855E8A7C-ECB4-4CA3-B045-1DFA50104289
```

Essa fonte corresponde ao serviço `Windows Store (Prod)`, usado pela Store para localizar e instalar pacotes modernos.

O Windows Update, por sua vez, falhava anteriormente com `0x80072F8F`, causado por falhas de confiança TLS. A análise da cadeia de certificados mostrou problemas de revogação (`0x80092013`). A causa local encontrada foi a chave:

```text
HKLM\SOFTWARE\Microsoft\SystemCertificates\AuthRoot\AutoUpdate
```

Essa chave estava com herança de permissões desabilitada e timestamps de sincronização parados em `20/03/2024`. Como consequência, o serviço `CryptSvc` não conseguia atualizar corretamente os metadados de certificados raiz, certificados não permitidos e regras de confiança.

Com a validação de certificados quebrada, o Windows Update não conseguia sincronizar metadados SLS/DCAT. Isso impedia o registro correto das fontes usadas pela Microsoft Store e resultava no erro final `0x80248014`.

Além disso, após a correção da cadeia de certificados e do registro dos serviços DCAT, ainda havia uma tentativa antiga de instalação presa na fila da Store. Essa fila precisava ser cancelada antes de uma nova instalação funcionar.

## Causa raiz

A causa raiz foi uma falha local na atualização de confiança de certificados do Windows, provocada por permissões incorretas na chave `AuthRoot\AutoUpdate`.

Essa falha impediu o `CryptSvc` de manter atualizadas informações usadas pela CryptoAPI. Com isso, o Windows Update passou a falhar na validação TLS dos serviços da Microsoft, e a Microsoft Store deixou de conseguir usar corretamente os serviços federados DCAT necessários para instalação de aplicativos.

## Correção aplicada

As ações realizadas foram:

1. Verificação inicial dos pacotes da Microsoft Store, App Installer e serviços relacionados.
2. Identificação do erro `0x80248014` durante a instalação pela Store.
3. Verificação direta do Windows Update, que retornou `0x80072F8F`.
4. Testes TLS/Schannel nos endpoints da Microsoft, revelando erro de revogação `0x80092013`.
5. Ajuste para o Windows preferir IPv4 em conexões dual-stack, pois alguns endpoints Microsoft estavam travando durante a negociação TLS via IPv6.
6. Execução de reparo da imagem do sistema com `DISM /Online /Cleanup-Image /RestoreHealth`.
7. Execução de verificação de integridade com `sfc /scannow`.
8. Restauração da herança de permissões em `AuthRoot\AutoUpdate`, permitindo novamente acesso do serviço `NT SERVICE\CryptSvc`.
9. Atualização dos timestamps locais de sincronização de certificados raiz, certificados não permitidos e regras de pinagem.
10. Reinicialização dos serviços `CryptSvc`, `wuauserv`, `BITS`, `InstallService`, `ClipSVC`, `AppXSVC` e `wlidsvc`.
11. Registro das fontes federadas do Windows Update usadas pela Store:

```text
Microsoft Update
DCat Flighting Prod
Windows Store (Prod)
Windows Update
```

12. Confirmação de que o Windows Update voltou a pesquisar atualizações com sucesso.
13. Cancelamento do item antigo de instalação do WhatsApp preso na fila interna da Store.
14. Nova instalação do pacote pela Microsoft Store via `winget`.

## Resultado

Após a correção, o Windows Update voltou a consultar os serviços da Microsoft corretamente e a Microsoft Store passou a resolver a fonte `Windows Store (Prod)` sem falhas.

O WhatsApp foi instalado com sucesso pela Store:

```text
Nome: WhatsApp
ID: 9NKSQGP7F2NH
Versão: 2.2616.100.0
Pacote: 5319275A.WhatsAppDesktop_2.2616.100.0_x64__cv1g1gvanyjgm
Status: Ok
```

## Observação final

O problema não estava no WhatsApp. A tentativa de instalá-lo apenas revelou uma falha acumulada na infraestrutura local de certificados, Windows Update e Microsoft Store. A solução efetiva foi restaurar a capacidade do Windows de validar os serviços da Microsoft, registrar novamente as fontes DCAT necessárias e limpar a fila de instalação que havia ficado presa com o erro anterior.
