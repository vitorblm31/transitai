# olho_vivo.py — coleta e análise de bunching

Script de linha de comando que coleta posições de ônibus na API Olho Vivo (SPTrans) e mede, numa parada de referência, o intervalo entre passagens (*headway*) e a taxa de agrupamento (*bunching*). É a base técnica para construir o alvo do modelo do TransitAI.

> **Status:** a etapa de **análise** foi testada com dados sintéticos. A etapa de **coleta** não foi validada contra a API real: em 07/10/2026 o login (`/Login/Autenticar`) respondeu `false` com duas chaves cadastradas. Os endpoints `Linha/Buscar` e `Parada/BuscarParadasPorLinha` seguem a documentação oficial, mas não foram exercitados ao vivo.

## Visão geral

```
Olho Vivo (tempo real)            disco                      análise
┌──────────────────┐   coletar   ┌───────────────┐  analisar  ┌──────────────────────────┐
│ /Posicao/Linha   │ ──────────▶ │ posicoes.csv  │ ─────────▶ │ passagens na parada      │
│ /Parada/...      │ ──────────▶ │ paradas.csv   │ ─────────▶ │ headways, bunching, gaps │
└──────────────────┘             └───────────────┘            └──────────────────────────┘
```

A API só devolve o estado atual da frota e não guarda histórico, então o coletor precisa ficar rodando pelo período que se quer analisar.

## Requisitos

- Python 3.8 ou superior.
- `requests` (coleta) e `pandas` (análise).

```bash
pip install requests pandas
```

## Configuração

| Variável | Obrigatória | Padrão | Função |
|---|---|---|---|
| `SPTRANS_TOKEN` | sim | — | Token da API, gerado em *Meus Aplicativos* no portal de desenvolvedores |
| `SPTRANS_BASE` | não | `https://api.olhovivo.sptrans.com.br/v2.1` | URL base da API |

O token é lido do ambiente e **nunca** deve ser escrito no código nem versionado. Cada terminal tem as suas próprias variáveis.

```bash
# Linux/Mac
export SPTRANS_TOKEN="seu_token"
# Windows (PowerShell): aspas, sem espaço depois do "="
$env:SPTRANS_TOKEN="seu_token"
```

A documentação oficial informa que o acesso por HTTP foi descontinuado: use `https`.

## Comandos

### `buscar` — achar o código da linha

```bash
python olho_vivo.py buscar 8000
```

Lista, para cada linha que casa com o termo: `cl` (código), `lt` (letreiro), sentido (`sl`), terminal principal (`tp`) e secundário (`ts`). **Cada `cl` corresponde a um sentido.**

### `coletar` — gravar posições

```bash
python olho_vivo.py coletar --cl 12345 67890 --intervalo 45 --minutos 180 --saida dados
```

| Argumento | Padrão | Significado |
|---|---|---|
| `--cl` | obrigatório | Um ou mais códigos de linha |
| `--intervalo` | 45 | Segundos entre rodadas de coleta |
| `--minutos` | 180 | Duração total da coleta |
| `--saida` | `dados` | Pasta de saída |

Na primeira execução, grava `paradas.csv` com as paradas de cada linha (para escolher o ponto de referência). A cada rodada, consulta `/Posicao/Linha` de cada `cl` e **acrescenta** linhas a `posicoes.csv`; rodar de novo continua o mesmo arquivo. Falhas de rede em uma consulta geram um aviso em `stderr` e não interrompem a coleta.

### `analisar` — headway e bunching

```bash
python olho_vivo.py analisar --dados dados --cl 12345 --parada 700012345 --headway-prog 6 --raio 60
```

| Argumento | Padrão | Significado |
|---|---|---|
| `--dados` | `dados` | Pasta com os CSVs da coleta |
| `--cl` | obrigatório | Linha/sentido a analisar |
| `--parada` | obrigatório | `cp` da parada de referência (ver `paradas.csv`) |
| `--headway-prog` | obrigatório | Intervalo programado, em minutos (da grade ou do GTFS) |
| `--raio` | 60 | Distância máxima, em metros, para contar uma passagem |

Imprime o número de passagens, mediana/mínimo/máximo do headway observado, a taxa de bunching (`headway < 50%` do programado) e a de buracos (`headway > 150%` do programado). Salva `headways_<cl>_<parada>.csv` na pasta de dados.

## Formato dos dados

**`posicoes.csv`**

| Coluna | Origem na API | Descrição |
|---|---|---|
| `cl` | parâmetro da consulta | Código da linha/sentido |
| `p` | `p` | Prefixo do veículo |
| `ta` | `ta` | Horário da captura, ISO 8601 em UTC |
| `py`, `px` | `py`, `px` | Latitude, longitude |
| `a` | `a` | Acessibilidade (gravada, não usada na análise) |

**`paradas.csv`:** `cl`, `cp` (código da parada), `np` (nome), `py`, `px`.

**`headways_<cl>_<parada>.csv`:** `passagem_brt` (instante da passagem do veículo seguinte, em horário de Brasília) e `headway_min` (minutos desde a passagem anterior).

## Como a análise funciona

1. **Preparação.** Filtra a linha, remove duplicatas por `(p, ta)`, converte `ta` de UTC para Brasília (UTC−3, deslocamento fixo) e ordena por veículo e tempo.
2. **Projeção plana.** Converte latitude/longitude em metros por uma projeção equirretangular local, centrada na parada. Para distâncias de algumas centenas de metros, o erro é desprezível.
3. **Detecção de passagem.** Para cada par de amostras consecutivas de um mesmo veículo, calcula a menor distância entre a parada e o segmento que une as duas posições. Se for menor ou igual a `--raio`, a passagem ocorre no instante interpolado linearmente ao longo do segmento. Pares separados por mais de 180 s ou com tempo não positivo são descartados, pois não dá para interpolar com confiança.
4. **Uma passagem por veículo.** Detecções do mesmo veículo a menos de 300 s uma da outra são tratadas como a mesma passagem (a parada costuma cair em dois segmentos adjacentes); fica a de menor distância.
5. **Headways.** Ordena todas as passagens no tempo e toma as diferenças sucessivas, em minutos. São necessárias pelo menos 3 passagens.
6. **Métricas.** Bunching: `h < 0,5 × prog`. Buracos: `h > 1,5 × prog`. A mediana é o elemento central da lista ordenada (`sorted(hw)[n // 2]`).

### Teste sintético

A análise foi verificada com dados gerados: 12 ônibus passando a 20 km/h por uma parada, com intervalo de 6 min e dois pares agrupados (1 min de diferença entre eles), amostrados a cada 45 s. O resultado esperado e obtido foi 12 passagens e 2 de 11 intervalos em bunching (18,2%), além de 2 de 11 em buraco (18,2%). Esse teste achou um erro real: sem a regra de uma passagem por veículo, cada passagem era contada duas vezes e o bunching saía inflado (60,9%).

```python
import csv, math, random
from datetime import datetime, timedelta, timezone
random.seed(1)
lat0, lon0 = -23.55, -46.65
with open("paradas.csv", "w", newline="") as f:
    w = csv.writer(f); w.writerow(["cl", "cp", "np", "py", "px"]); w.writerow([111, 999, "ref", lat0, lon0])
t0 = datetime(2026, 10, 7, 15, 0, tzinfo=timezone.utc)
rows, v = [], 20000 / 3600
for k, s in enumerate([0, 6, 12, 13, 24, 30, 36, 37, 48, 54, 60, 66]):
    tp = t0 + timedelta(minutes=s + 10)
    for step in range(-12, 13):
        ts = tp + timedelta(seconds=45 * step + random.randint(-3, 3))
        lon = lon0 + v * (ts - tp).total_seconds() / (111320 * math.cos(math.radians(lat0)))
        rows.append([111, f"B{k}", ts.isoformat().replace("+00:00", "Z"), lat0 + 0.00001, lon, True])
with open("posicoes.csv", "w", newline="") as f:
    w = csv.writer(f); w.writerow(["cl", "p", "ta", "py", "px", "a"]); w.writerows(rows)
```

```bash
python olho_vivo.py analisar --dados . --cl 111 --parada 999 --headway-prog 6
```

## Limitações conhecidas

- **Headway programado único.** `--headway-prog` é um valor constante informado à mão; a grade real varia ao longo do dia. Ainda não há leitura do GTFS.
- **Uma linha/sentido por análise.** Variantes de uma mesma linha com `cl` diferentes não são combinadas.
- **Sem renovação de sessão.** O login é feito uma vez no início; se a sessão expirar durante uma coleta longa, as consultas passam a falhar com aviso e é preciso reiniciar.
- **Amostragem.** Com intervalo de 45 s e ônibus a ~20 km/h, o veículo avança cerca de 250 m entre amostras; a interpolação lida com isso, mas amostras muito espaçadas (acima de 180 s) perdem passagens.
- **Passagens repetidas.** Um veículo que passe duas vezes pela mesma parada em menos de 5 min (percurso em laço) é contado uma vez.
- **Limiares de bunching.** 50% e 150% do programado são hipóteses de trabalho, sem definição oficial da SPTrans.
- **Não há tratamento de resposta fora do padrão.** Respostas não JSON caem no aviso genérico de falha.
- **Escopo do dado.** A API não traz passageiros; o script usa só posição e tempo.

## Solução de problemas

| Sintoma | Causa provável | O que fazer |
|---|---|---|
| `Defina SPTRANS_TOKEN no ambiente.` | Variável não definida neste terminal | Definir de novo; cada terminal é uma sessão separada |
| `Falha na autenticação` | A API devolveu algo diferente de `true` | Conferir tamanho do token (`$env:SPTRANS_TOKEN.Length`, esperado 64), tirar espaços (`.Trim()`), usar `https`; se persistir, a chave pode não estar valendo e é preciso contatar a SPTrans |
| Nenhuma saída ao rodar | Arquivo incompleto ou colado com erro | Conferir se termina em `a = ap.parse_args(); a.f(a)` e rodar `--help` |
| `IndentationError` | Indentação alterada ao colar | Alinhar a linha indicada com as vizinhas |
| `can't open file` / erro de caminho no VS Code | Botão *Run* usa o interpretador errado e não passa argumentos | Rodar pelo terminal integrado, na pasta do script |
| `HTTP 411 Length Required` no `curl` | POST sem corpo | Adicionar `-H "Content-Length: 0"`; o Python envia o cabeçalho sozinho |
| `Passagens insuficientes` | Pouco tempo de coleta ou parada fora do trajeto | Coletar por mais tempo, aumentar `--raio` ou escolher outra parada |

## Próximos passos técnicos (previstos)

1. Resolver o acesso à API e validar `coletar` com dados reais.
2. Ler o GTFS (`stop_times.txt`) para obter o headway programado por faixa horária.
3. Montar o alvo em janelas de 30 min por linha × sentido (por exemplo, `y = 1` se houver bunching na janela seguinte) e as variáveis calculadas só até o instante de corte.
4. Treinar um modelo de *gradient boosting* (LightGBM) com divisão temporal e compará-lo a um baseline que ordena as linhas pelo desvio atual de intervalo, medindo precision@k e recall@k.
5. Adicionar renovação automática de sessão e testes automatizados.
