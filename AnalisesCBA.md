## **Primeira Análise: Contagem_Velocidade_Por_Equipamento**

### **Configuração de Agendamento:**
- **Schedule:** Periodic
- **Period:** 5 minutes

### **Expressões (cada linha é uma expressão separada):**

| Linha | Variable/Output | Expression |
|-------|----------------|------------|
| 1 | Limite | 3 |
| 2 | TempoInicio | "*-1h" |
| 3 | TempoFim | "*" |
| 4 | ContagemRD04 | EventCount('Velocidade Horizontal RD04', TempoInicio, TempoFim, ">3") |
| 5 | ContagemRD06 | EventCount('Velocidade Horizontal RD06', TempoInicio, TempoFim, ">3") |

### **Mapeamento de Outputs:**
- ContagemRD04 → 'Contagem RD04 Ultima Hora'
- ContagemRD06 → 'Contagem RD06 Ultima Hora'

---

## **Segunda Análise: Tendencia_Velocidade_Individual**

### **Configuração de Agendamento:**
- **Schedule:** Periodic  
- **Period:** 1 hour

### **Expressões - Parte 1 (Análise RD04):**

| Linha | Variable | Expression |
|-------|----------|------------|
| 1 | DadosRD04 | RecordedValues('Contagem RD04 Ultima Hora', "*-48h", "*") |
| 2 | NumValoresRD04 | Count(DadosRD04) |
| 3 | SomaRD04 | TagTotal('Contagem RD04 Ultima Hora', "*-48h", "*") |
| 4 | MediaRD04 | SomaRD04 / NumValoresRD04 |
| 5 | UltimoRD04 | TagVal('Contagem RD04 Ultima Hora', "*") |
| 6 | RD04_24h | TagAvg('Contagem RD04 Ultima Hora', "*-24h", "*") |
| 7 | RD04_48h | TagAvg('Contagem RD04 Ultima Hora', "*-48h", "*-24h") |
| 8 | CrescRD04_24h | RD04_24h - RD04_48h |
| 9 | CrescRD04_Recente | UltimoRD04 - RD04_24h |
| 10 | TaxaCrescRD04 | CrescRD04_Recente |

### **Expressões - Parte 2 (Análise RD06):**

| Linha | Variable | Expression |
|-------|----------|------------|
| 11 | DadosRD06 | RecordedValues('Contagem RD06 Ultima Hora', "*-48h", "*") |
| 12 | NumValoresRD06 | Count(DadosRD06) |
| 13 | SomaRD06 | TagTotal('Contagem RD06 Ultima Hora', "*-48h", "*") |
| 14 | MediaRD06 | SomaRD06 / NumValoresRD06 |
| 15 | UltimoRD06 | TagVal('Contagem RD06 Ultima Hora', "*") |
| 16 | RD06_24h | TagAvg('Contagem RD06 Ultima Hora', "*-24h", "*") |
| 17 | RD06_48h | TagAvg('Contagem RD06 Ultima Hora', "*-48h", "*-24h") |
| 18 | CrescRD06_24h | RD06_24h - RD06_48h |
| 19 | CrescRD06_Recente | UltimoRD06 - RD06_24h |
| 20 | TaxaCrescRD06 | CrescRD06_Recente |

### **Expressões - Parte 3 (Detecção de Tendências):**

| Linha | Variable/Output | Expression |
|-------|----------------|------------|
| 21 | LimiteCrescimento | 5 |
| 22 | LimiteTaxa | 0.5 |
| 23 | AceleracaoRD04 | If CrescRD04_24h > 0 Then (CrescRD04_Recente - CrescRD04_24h) / CrescRD04_24h Else 0 |
| 24 | AceleracaoRD06 | If CrescRD06_24h > 0 Then (CrescRD06_Recente - CrescRD06_24h) / CrescRD06_24h Else 0 |
| 25 | AlertaRD04_Cond1 | CrescRD04_Recente > LimiteCrescimento |
| 26 | AlertaRD04_Cond2 | AceleracaoRD04 > LimiteTaxa |
| 27 | AlertaRD04_Cond3 | CrescRD04_24h > 2 AND CrescRD04_Recente > 2 |
| 28 | TendenciaRD04 | AlertaRD04_Cond1 OR AlertaRD04_Cond2 OR AlertaRD04_Cond3 |
| 29 | AlertaRD06_Cond1 | CrescRD06_Recente > LimiteCrescimento |
| 30 | AlertaRD06_Cond2 | AceleracaoRD06 > LimiteTaxa |
| 31 | AlertaRD06_Cond3 | CrescRD06_24h > 2 AND CrescRD06_Recente > 2 |
| 32 | TendenciaRD06 | AlertaRD06_Cond1 OR AlertaRD06_Cond2 OR AlertaRD06_Cond3 |

### **Mapeamento de Outputs:**
- TendenciaRD04 → 'Tendencia RD04 Detectada'
- TendenciaRD06 → 'Tendencia RD06 Detectada'
- TaxaCrescRD04 → 'Taxa Crescimento RD04'
- TaxaCrescRD06 → 'Taxa Crescimento RD06'
- AceleracaoRD04 → 'Aceleracao RD04'
- AceleracaoRD06 → 'Aceleracao RD06'

---

## **Versão Simplificada Alternativa (Mais Fácil de Implementar)**

Se você estiver tendo problemas com a versão acima, aqui está uma versão mais simples:

### **Análise Simplificada de Tendência:**

| Linha | Variable/Output | Expression |
|-------|----------------|------------|
| 1 | ValorAtualRD04 | 'Contagem RD04 Ultima Hora' |
| 2 | ValorAnteriorRD04 | PrevVal('Contagem RD04 Ultima Hora', "*-1h") |
| 3 | Media24hRD04 | TagAvg('Contagem RD04 Ultima Hora', "*-24h", "*") |
| 4 | CrescimentoRD04 | ValorAtualRD04 - ValorAnteriorRD04 |
| 5 | TendenciaRD04 | ValorAtualRD04 > Media24hRD04 * 1.3 AND CrescimentoRD04 > 0 |
| 6 | ValorAtualRD06 | 'Contagem RD06 Ultima Hora' |
| 7 | ValorAnteriorRD06 | PrevVal('Contagem RD06 Ultima Hora', "*-1h") |
| 8 | Media24hRD06 | TagAvg('Contagem RD06 Ultima Hora', "*-24h", "*") |
| 9 | CrescimentoRD06 | ValorAtualRD06 - ValorAnteriorRD06 |
| 10 | TendenciaRD06 | ValorAtualRD06 > Media24hRD06 * 1.3 AND CrescimentoRD06 > 0 |

### **Dicas para Inserir no PI AF:**

1. **No PI System Explorer**, ao criar a análise:
   - Cada linha da tabela deve ser inserida como uma linha separada
   - Use o botão "+" para adicionar novas linhas
   - Digite primeiro o nome da variável, depois a expressão

2. **Para referenciar atributos:**
   - Use aspas simples: 'Nome do Atributo'
   - Ou use o botão de browse para selecionar

3. **Para funções de tempo:**
   - Use aspas duplas para timestamps: "*-1h", "*-24h"
   - O asterisco (*) significa "agora"

4. **Ordem importante:**
   - Insira as linhas na ordem mostrada
   - Variáveis devem ser definidas antes de serem usadas
