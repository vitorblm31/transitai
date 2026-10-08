# TransitAI

Modelagem preditiva de agrupamento de ônibus (*bunching*) para apoiar a regulação operacional de frotas urbanas, com a SPTrans (São Paulo) como organização-alvo.

Projeto aplicado da disciplina **Aprendizagem de Máquina Supervisionada (MLS 2026.2)**, Ciência de Dados para Negócios, UFPB/CCSA.

**Equipe:** Vítor Batista, Marcello Siqueira, Ricardo César.

## Problema

Quando duas ou mais viagens da mesma linha chegam juntas a um ponto, o passageiro espera mais e a regularidade do serviço piora. Hoje o controle operacional reage a ocorrências já visíveis. A pergunta do projeto é: **dá para prever, a cada janela de tempo, quais linhas vão agrupar, para que a equipe atue antes?**

A decisão a apoiar é *em quais linhas ou corredores intervir*, com capacidade limitada de intervenções por janela. O modelo deve gerar um ranking, e a utilidade é medida por precision@k e recall@k, além do custo esperado dos erros.

## Status

A equipe **não realizou escuta nem entrevista** com a SPTrans. O documento da Entrega 3 é, por isso, um cenário hipotético com um Canvas de ML, e cada afirmação é marcada como:

- **[F]** fato verificável em fonte pública citada;
- **[H]** hipótese da equipe, sem confirmação;
- **[L]** lacuna a resolver com o demandante.

Parâmetros como a capacidade de intervenção (k = 2), a cadência de decisão (30 min) e a razão entre custos dos erros (C/B = 3) são **hipóteses de trabalho**, não dados da SPTrans.

A coleta própria de posições ainda não foi concluída: em 07/10/2026 o login da API Olho Vivo respondeu `false` com duas chaves cadastradas, e o contato com a SPTrans está pendente. Nenhum resultado medido é citado nos documentos.

## Recorte

- **Unidade de análise:** linha × sentido × janela de 30 min.
- **Alvo (hipótese):** bunching na janela seguinte, definido como intervalo observado abaixo de 50% do programado.
- **Corte temporal:** o modelo usa só observações até o instante *t* para prever a janela de *t* a *t* + 30 min.
- **Corredores:** Santo Amaro/Nove de Julho, Paes de Barros e Centro-Lapa-Pirituba.
- **Modelagem prevista:** *gradient boosting* (LightGBM), comparado a um baseline operacional simples (ordenar pelo desvio atual de intervalo).

## Fontes de dados

| Fonte | O que traz | Observação |
|---|---|---|
| API Olho Vivo (SPTrans) | Posição em tempo real: prefixo, lat/long, acessibilidade, timestamp (UTC), linha, sentido, previsão de chegada | Exige token; **não traz dados de passageiros**; não guarda histórico |
| GTFS estático (SPTrans) | Linhas, paradas, horários programados | Exige cadastro no portal de desenvolvedores |
| CGE-SP | Chuva (pluviômetros em 32 subprefeituras) | Sem API pública confirmada; distribuição por planilhas |

As fontes completas, com links e datas de acesso, estão no PDF da Entrega 3.

## Estrutura do repositório

```
transitai/
├── README.md
├── docs/
│   └── E03_equipeXX_escuta_canvas.pdf   # Entrega 3: cenário hipotético e Canvas de ML
├── src/
│   └── olho_vivo.py                     # coleta e análise de bunching (Olho Vivo)
└── .gitignore                           # ignora dados/ e credenciais
```

## Como rodar a coleta

Requer Python 3 e as bibliotecas `requests` e `pandas`:

```bash
pip install requests pandas
```

O token é gerado em **Meus Aplicativos**, no portal de desenvolvedores da SPTrans. Ele é lido da variável de ambiente `SPTRANS_TOKEN` e **nunca deve ser escrito no código nem versionado**.

```bash
# Linux/Mac
export SPTRANS_TOKEN="seu_token"
# Windows (PowerShell)
$env:SPTRANS_TOKEN="seu_token"
```

A API responde apenas por HTTPS. A URL base padrão é `https://api.olhovivo.sptrans.com.br/v2.1` e pode ser trocada com a variável `SPTRANS_BASE`.

```bash
# 1. achar o código (cl) da linha; cada cl corresponde a um sentido
python src/olho_vivo.py buscar 8000

# 2. coletar posições (a API é em tempo real, então é preciso deixar rodando)
python src/olho_vivo.py coletar --cl 12345 --intervalo 45 --minutos 180 --saida dados

# 3. medir headway observado e taxa de bunching numa parada de referência
python src/olho_vivo.py analisar --dados dados --cl 12345 --parada 700012345 --headway-prog 6
```

O intervalo programado (`--headway-prog`, em minutos) deve ser lido do GTFS ou da grade de horários da linha. O script foi testado apenas com dados sintéticos na etapa de análise; a coleta ainda não foi validada contra a API.

## Ética e LGPD

- A API não traz dados de passageiros, mas o prefixo do veículo pode ser ligado a motoristas se cruzado com escalas. Esse cruzamento não faz parte do projeto.
- Uso vedado proposto: avaliar ou punir motoristas individualmente.
- Não versionar tokens, dados brutos coletados ou qualquer dado pessoal.

## Declaração de uso de IA generativa

Foi usado o Claude (Anthropic) como apoio na estruturação e redação dos documentos, na pesquisa de fontes e na geração do script de coleta. A equipe responde pela verificação de tudo o que está neste repositório.
