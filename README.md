# Transmissão EFD-REINF (Série R-4000) via Token A3

## ⚠️ AVISO IMPORTANTE PARA A IA
**NÃO altere ou analise a lógica de geração e assinatura dos lotes XML.** 
A rotina de extração de dados e a injeção da assinatura digital (`<ds:Signature>`) via XMLDSig já foram exaustivamente testadas e estão 100% funcionais. Os arquivos na pasta `/envios` já possuem validade jurídica.
**O foco exclusivo deste projeto é a camada de REDE: a transmissão e consulta dos arquivos assinados superando barreiras de firewall e proxy.**

## 📌 Contexto
Este módulo é responsável por pegar lotes XML validados e transmiti-los para o webservice da Receita Federal utilizando um certificado digital físico (Token A3). 

## 🛠️ Arquitetura de Transmissão
* O Python atua apenas como orquestrador de pastas.
* O "trabalho pesado" de rede e criptografia é delegado ao **PowerShell**, que atua como ponte com o hardware (Token A3), disparando nativamente a janela de PIN do Windows e estabelecendo o túnel SSL/TLS.
* Foi adotada a técnica de usar **`System.Net.HttpWebRequest`** diretamente (em vez de `Invoke-WebRequest`) para ter controle total sobre o handshake TLS, Keep-Alive e ServicePoint.

## 🔧 Mudanças Arquiteturais (Refatoração 2025-01)

### Problema Resolvido
O erro **"A conexão subjacente estava fechada: Erro inesperado em um recebimento"** era causado por 5 fatores combinados:

| # | Causa | Solução |
|---|-------|---------|
| 1 | `MaxServicePointIdleTime` padrão de 100ms matava a conexão antes da resposta | Configurado para 300000ms (5 min) |
| 2 | `Invoke-WebRequest` não permitia controle fino do TLS/Keep-Alive | Substituído por `HttpWebRequest` direto |
| 3 | Body enviado como string corrompia a assinatura ou adicionava BOM | Envio como `byte[]` via `RequestStream` |
| 4 | Ausência de headers SOAP (`SOAPAction`, `Accept`) | Headers adicionados ao request |
| 5 | Proxy corporativo não configurado em redes de órgãos públicos | `PROXY_URL` configurável no `config.py` |

### Outras Melhorias
- **Retry automático**: 3 tentativas com backoff de 5s entre cada
- **Timeout estendido**: 120s para transmissão, 90s para consulta (servidor da Receita é lento)
- **XML de consulta via arquivo temporário**: Evita problemas de escape de aspas no PowerShell
- **Diagnóstico aprimorado**: Log do certificado encontrado (Subject, Thumbprint, Vencimento)
- **Tratamento de erro granular**: Distingue HTTP 422/500 de falha de conexão

## 🚀 Como Executar
1. Garanta que os arquivos `.xml` assinados estejam na pasta `envios/`.
2. O Token A3 (Autoridade Certificadora da Defesa) deve estar conectado.
3. Se estiver em rede corporativa com proxy, configure `PROXY_URL` no `config.py`.
4. Execute: `python transmissao_a3.py`

## 📁 Estrutura de Diretórios
```
C:\Users\EFD-REINF\
│
├── envios/               # Arquivos XML gerados e ASSINADOS (Aguardando transmissão)
├── recebidos/            # Arquivos XML transmitidos com sucesso (Movidos para cá após o recibo)
├── protocolos/           # Respostas da Receita Federal (Recibos de sucesso ou logs de erro/HTTP 422)
│
├── config.py             # Variáveis globais (URLs, CNPJ, Caminhos, Proxy, Timeouts)
├── transmissao_a3.py     # Script principal de transmissão (refatorado)
├── consulta_lotes.py     # Script de consulta de protocolo (refatorado)
├── requirements.txt      # Dependências do Python
├── layout_exemplo.md     # Mapeamento do tabelão do SIAFI
├── log_erro_exemplo.txt  # Análise técnica do erro original e causa raiz
└── documentacao_tecnica.txt # Restrições de rede e hardware
```

## ⚙️ Configuração do Proxy
Se estiver em rede corporativa (órgão público), edite o `config.py`:
```python
# Sem proxy:
PROXY_URL = ""

# Com proxy:
PROXY_URL = "http://usuario:senha@proxy.orgao.gov.br:8080"
```

## 🔐 Configuração do Certificado
O filtro do certificado é configurável no `config.py`:
```python
CERT_SUBJECT_FILTER = "*NOME NO CERTIFICADO*"
```
Para descobrir o nome exato, abra o PowerShell e execute:
```powershell
Get-ChildItem -Path Cert:\CurrentUser\My | Select-Object Subject
```

## 📋 Detalhes Técnicos da Refatoração

### Antes (quebrado):
```powershell
# Invoke-WebRequest sem controle de TLS
$response = Invoke-WebRequest -Uri $url -Method Post -Body $xml -Certificate $cert
```

### Depois (corrigido):
```powershell
# HttpWebRequest com controle total
$req = [System.Net.HttpWebRequest]::Create($url)
$req.KeepAlive = $true                                          # ← MANTÉM CONEXÃO VIVA
$req.ClientCertificates.Add($cert)                              # ← CERTIFICADO A3
[Net.ServicePointManager]::MaxServicePointIdleTime = 300000     # ← 5 MINUTOS
$req.Headers.Add("SOAPAction", "...")                           # ← HEADER OBRIGATÓRIO
# Body enviado como byte[] via RequestStream                     # ← PRESERVA ASSINATURA
```

### Por que HttpWebRequest em vez de Invoke-WebRequest?
| Aspecto | Invoke-WebRequest | HttpWebRequest |
|---------|-------------------|----------------|
| KeepAlive | Não configurável | `$req.KeepAlive = $true` |
| ServicePoint | Sem acesso | `MaxServicePointIdleTime` configurável |
| Body encoding | String (pode corromper) | byte[] (preserva assinatura) |
| Client Certificate | `-Certificate` limitado | `.ClientCertificates.Add()` com controle |
| SOAP Headers | Via `-Headers` limitado | `.Headers.Add()` completo |
| Timeout | `-TimeoutSec` básico | `.Timeout` + `.ReadWriteTimeout` granular |