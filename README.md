Resumo Técnico: Workflow n8n & RPA de
Investimentos
1. Atualização do Script Python (RPA de Ingestão)
O script Python foi desenvolvido para realizar a extração (scraping) da tabela de clientes, converter
o saldo de formato textual (ex: "R$ 1.500,00") para número do tipo float e enviar todos os registros
como um payload JSON ao Webhook do n8n.
import requests
from bs4 import BeautifulSoup
URL_PAGINA = "https://digitalinnovationone.github.io/dio-lab-assistente-investimentos-rpan8n/"
N8N_WEBHOOK = "http://localhost:5678/webhook-test/cliente"
html = requests.get(URL_PAGINA).text
soup = BeautifulSoup(html, "html.parser")
linhas = soup.select("#clientes tbody tr")
clientes = []
for linha in linhas:
colunas = linha.find_all("td")
saldo_raw = colunas[2].get_text(strip=True)
saldo_limpo = (
saldo_raw.replace("R$", "")
.replace(".", "")
.replace(",", ".")
.strip()
)
cliente = {
"nome": colunas[0].get_text(strip=True),
"email": colunas[1].get_text(strip=True),
"saldo": float(saldo_limpo) if saldo_limpo else 0.0,
"perfil": colunas[3].get_text(strip=True)
}
clientes.append(cliente)
payload = {"clientes": clientes}
try:
response = requests.post(N8N_WEBHOOK, json=payload)
print(f"Status do Envio: {response.status_code}")
print("Resposta do n8n:", response.text)
except Exception as e:
print(f"Erro ao conectar com o n8n: {e}")
2. Arquitetura do Workflow no n8n
Estrutura de 4 Etapas Encadeadas:
1. Webhook Node: Recebe o payload JSON contendo a lista de clientes enviada pelo script
Python.
2. Code Node (Desmembrar Clientes / Webhook Data): Processa os itens recebidos e
transforma em execuções individuais.
Documento Técnico - Workflow n8n Página 1
3. Código do Nó: Analista de Perfil do Investidor
Este nó em JavaScript realiza o processamento do arquivo CSV recebido via HTTP Request e executa
o cruzamento direto com a lista de clientes vinda do Webhook, classificando os produtos elegíveis
para cada cliente:
// 1. Processa o CSV vindo do HTTP Request (entrada principal do nó)
const csvData = $input.first().json.data || '';
const linhasCsv = csvData.trim().split('\n');
linhasCsv.shift(); // Remove o cabeçalho
const matrizInvestimentos = linhasCsv.map(linha => {
const colunas = linha.split(',');
return {
perfil: colunas[0]?.trim(),
produto: colunas[1]?.trim(),
minimo: Number(colunas[2]?.trim()) || 0,
rentabilidade: colunas[3]?.trim()
};
});
// 2. Resgata TODOS os itens de clientes vindos do nó Webhook
const itensClientes = $('Webhook').all();
// 3. Itera sobre cada cliente capturando os campos reais e fazendo a analogia
return itensClientes.map(item => {
const cliente = item.json;
const nome = cliente.nome || cliente.cliente?.nome || 'Cliente';
const email = cliente.email || cliente.cliente?.email || '';
const perfil = cliente.perfil || cliente.cliente?.perfil || 'Conservador';
let saldoBruto = cliente.saldo !== undefined ? cliente.saldo : cliente.cliente?.saldo;
let saldoLimpo = 0;
if (typeof saldoBruto === 'number') {
saldoLimpo = saldoBruto;
} else if (typeof saldoBruto === 'string') {
saldoLimpo = Number(saldoBruto.replace('R$', '').replace(/\./g, '').replace(',',
'.').trim()) || 0;
}
const produtosRecomendados = matrizInvestimentos.filter(opcao =>
opcao.perfil.toLowerCase() === perfil.toLowerCase() && saldoLimpo >= opcao.minimo
);
return {
json: {
nome: nome,
email: email,
saldo: cliente.saldo || `R$ ${saldoLimpo.toFixed(2).replace('.', ',')}`,
perfil: perfil,
produtos_elegiveis: produtosRecomendados
}
};
});
4. Estruturação das Mensagens de Abordagem por Perfil
Documento Técnico - Workflow n8n Página 2
5. Prompt Principal para o Nó de IA (Geração de E-mail HTML)
Prompt do Nó Google Gemini (Message a Model):
Você é um Assessor de Investimentos sênior, especialista em comunicação por e-mail com
clientes de alta conversão e clareza.
Abaixo estão os dados do cliente, seu perfil de investidor e as 3 opções de estratégias
geradas pelo sistema para este cliente específico:
- Nome do Cliente: {{ $json.nome }}
- E-mail: {{ $json.email }}
- Saldo Disponível: {{ $json.saldo }}
- Perfil de Risco: {{ $json.perfil }}
- Opções Estratégicas Disponíveis:
1. {{ $json.opcoes_mensagem_email.opcao_1 }}
2. {{ $json.opcoes_mensagem_email.opcao_2 }}
3. {{ $json.opcoes_mensagem_email.opcao_3 }}
Sua tarefa é redigir um e-mail comercial altamente profissional, persuasivo e elegante
para o cliente, estruturando o conteúdo em HTML tabulado e limpo com quebras de linha adequadas e alinhado às
boas práticas de e-mail marketing.

DIRETRIZES OBRIGATÓRIAS:
1. Formatação HTML: Utilize uma estrutura de tabela limpa e responsiva para organizar o
layout (cabeçalho, saudação, corpo e rodapé).
2. Tom da Mensagem: Consultivo, empático, corporativo e focado em planejamento
financeiro.
3. Restrição Regulatória Absoluta: NUNCA indique, recomende ou cite a compra ou venda de
ações específicas de empresas (ex: evite citar tickers ou nomes de empresas como PETR4,
VALE3, etc.). Foque em classes de ativos genéricas.
4. Personalização: Utilize o nome do cliente, o saldo atual e o perfil de risco
informados nos dados acima.
5. Rodapé de Alerta de Risco: No final do e-mail, inclua obrigatoriamente um rodapé
destacado informando que todo e qualquer investimento envolve riscos, inclusive a
possibilidade de perda do capital investido, e que rentabilidade passada não é garantia
de rentabilidade futura.
Retorne APENAS o código HTML do e-mail pronto para ser disparado.

7. Resolução de Erros de Configuração e Conectividade

Erro / Problema Causa Identificada Solução Aplicada
1 - Referenced node doesn't exist
Referência incorreta a um nó inexistente

2 - Desmembrar Clientes.
Ajustado o código para referenciar diretamente o nó $('Webhook').all().

3 - Bad request - single turn requests end with user role
O campo Role nas configurações do nó do Gemini estava definido como Model.
Alterado o campo Role para User (ou deixado em branco).
