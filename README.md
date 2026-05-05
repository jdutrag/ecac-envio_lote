# ecac-envio_lote
Repositório de envio em lote de dados xml à receita federal por API REST dos eventos da série R-4020

Automação de Transmissão EFD-REINF (Série R-4000)

Este projeto tem como objetivo automatizar a transmissão de lotes de eventos da série R-4000 para a EFD-REINF, utilizando certificados digitais A3 (Token/Cartão) e garantindo a organização dos protocolos de recebimento para auditoria e conformidade.
1. Estrutura do Projeto

A organização dos arquivos segue um fluxo lógico para evitar o reprocessamento de lotes e facilitar a consulta posterior.
Plaintext

/EFD-REINF
│
├── /envios           # XMLs gerados (lotes de 50 eventos) aguardando transmissão.
├── /recebidos        # XMLs que foram transmitidos com sucesso.
├── /protocolos       # Retornos da Receita Federal (Protocolo de Envio/Recibo).
│
├── transmissao.py    # Script principal de comunicação via REST.
├── consulta.py       # Script para buscar o resultado do processamento.
└── manual_reinf.pdf  # Manual do Desenvolvedor (v2.7) utilizado como base.

2. Processo de Transmissão e Consulta

O projeto baseia-se nas orientações do Manual do Desenvolvedor da EFD-REINF, seguindo o modelo de comunicação assíncrona.
Lógica de Transmissão (transmissao.py)

    Varredura: O script identifica arquivos .xml na pasta /envios.

    Autenticação: Realiza o handshake SSL utilizando o certificado digital A3 (emitido pela Autoridade Certificadora da Defesa).

    Envio: Os lotes são enviados via método POST para o endpoint de recepção da Receita Federal.

    Protocolo: O ID do protocolo retornado é salvo na pasta /protocolos e o arquivo original é movido para /recebidos.

Lógica de Consulta (consulta.py)

    Leitura: O script lê os protocolos pendentes na pasta /protocolos.

    Verificação: Consulta o endpoint de processamento da Receita para verificar se o lote foi aceito ou se apresenta erros de validação.

    Finalização: Armazena o recibo definitivo de entrega.

3. Desafios Técnicos e Evolução do Modelo

O principal desafio deste projeto foi a comunicação entre o interpretador Python e o hardware do Certificado Digital A3.

    Modelo Inicial: Tentativa de uso da biblioteca requests com OpenSSL padrão. Este modelo resultou em Erro 496 (Certificado de Cliente Ausente), pois o Python não acessava nativamente a chave privada protegida pelo driver DXSafePKCS11.dll.

    Modelo de Integração (Schannel): Tentativa de forçar o Python a utilizar a camada de segurança do Windows (onde o certificado já está operacional) através da biblioteca pip-system-certs.

    Abordagem Atual: Para garantir a estabilidade em ambiente de produção (especialmente em organizações hierarquizadas e burocráticas), o modelo evoluiu para utilizar ferramentas nativas do sistema operacional que possuem acesso direto ao repositório de certificados do Windows.

4. Requisitos de Sistema

    Certificado Digital: Tipo A3 (Token ou Cartão) devidamente instalado e operacional no Windows.

    Drivers: Driver PKCS#11 compatível (ex: DXSafe para AC Defesa).

    Acesso à Rede: Liberação de firewall para os endpoints da Receita Federal nas portas 443.

    Nota de Contexto: Este projeto foi desenvolvido para atender às necessidades de execução financeira e orçamentária no âmbito da Marinha do Brasil, visando automatizar a transformação de dados provenientes do SIAFI em eventos válidos para a Receita Federal.
