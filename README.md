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
* Foi adotada a técnica de forçar a conexão via `.NET` (`Invoke-WebRequest` ou `System.Net.HttpWebRequest`) para superar limitações de redes corporativas restritas (como as de órgãos públicos).

## 🚀 Como Executar
1. Garanta que os arquivos `.xml` assinados estejam na pasta `envios/`.
2. O Token A3 (Autoridade Certificadora da Defesa) deve estar conectado.
3. Execute: `python transmissao_a3.py`


## Plaintext
C:\Users\EFD-REINF\
│
├── envios/               # Arquivos XML gerados e ASSINADOS (Aguardando transmissão)
├── recebidos/            # Arquivos XML transmitidos com sucesso (Movidos para cá após o recibo)
├── protocolos/           # Respostas da Receita Federal (Recibos de sucesso ou logs de erro/HTTP 422)
│
├── config.py             # Variáveis globais (URLs, CNPJ, Caminhos das pastas)
├── transmissao_a3.py     # Script principal (Foco da análise da IA)
├── requirements.txt      # Dependências do Python
├── layout_exemplo.md     # Mapeamento do tabelão do SIAFI
├── log_erro_exemplo.txt  # Exemplo do erro de Handshake que estamos enfrentando
└── documentacao_tecnica.txt # Restrições de rede e hardware
