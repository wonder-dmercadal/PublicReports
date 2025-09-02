# API REST - Sistema de Monitoramento de Barreiras Constellation

## Visão Geral

A API REST do Sistema de Monitoramento de Barreiras Constellation fornece acesso programático aos dados de monitoramento de barreiras de segurança em operações críticas de óleo e gás. Desenvolvida especificamente para integração com ferramentas de Business Intelligence como PowerBI, permite visualização em tempo real do status de barreiras operacionais.

### URL Base
```
https://api.constellation-barriers.com
```

### Características Principais

- Interface RESTful com respostas em formato JSON
- Autenticação via chave de API
- Suporte a paginação automática
- Filtros avançados por data, site e status
- Compatibilidade nativa com PowerBI
- Dados em tempo real com baixa latência
- Uptime garantido de 99.9%

---

## Autenticação

Todas as requisições à API requerem uma chave de autenticação fornecida no header `X-API-Key`.

### Como Autenticar

```http
GET /api/endpoint HTTP/1.1
Host: api.constellation-barriers.com
X-API-Key: sua-chave-de-api
```

### Exemplo com cURL

```bash
curl -H "X-API-Key: sua-chave-de-api" \
     https://api.constellation-barriers.com/api/dashboard-summary
```

### Códigos de Erro de Autenticação

| Status | Erro | Descrição |
|--------|------|-----------|
| 401 | Unauthorized | Chave de API ausente ou inválida |
| 403 | Forbidden | Acesso negado ao recurso solicitado |

---

## Endpoints

### Status da API

Verifica a disponibilidade do serviço.

**GET** `/health`

**Autenticação:** Não requerida

**Resposta:**
```json
{
  "status": "healthy",
  "timestamp": "2025-09-02T17:00:00Z",
  "version": "1.0"
}
```

---

### Resumo do Dashboard

Retorna estatísticas consolidadas para dashboards executivos.

**GET** `/api/dashboard-summary`

**Resposta:**
```json
{
  "summary": {
    "total_barriers": 9762,
    "problem_barriers": 11,
    "active_barriers": 9751,
    "degraded_barriers": 8,
    "inactive_barriers": 3,
    "monitoring_cycles": 17,
    "last_update": "2025-09-02T16:58:05Z"
  },
  "by_site": {
    "LGS": 1200,
    "ATS": 1100,
    "GDS": 950,
    "LNS": 900
  },
  "by_status": {
    "ATIVO": 9751,
    "DEGRADADA": 8,
    "INATIVA": 3
  },
  "notifications": {
    "total_sent": 16,
    "success_rate": 0.98
  }
}
```

---

### Histórico de Mudanças

Recupera o histórico de alterações de status das barreiras.

**GET** `/api/change-history`

**Parâmetros de Query:**

| Parâmetro | Tipo | Descrição | Exemplo |
|-----------|------|-----------|---------|
| `page` | integer | Número da página (padrão: 1) | `?page=2` |
| `per_page` | integer | Registros por página (padrão: 1000) | `?per_page=500` |

**Resposta:**
```json
{
  "data": [
    {
      "timestamp": "2025-09-02T10:00:00Z",
      "barrier_id": "LGS-BA-0053",
      "site": "LGS",
      "equipment": "LGS-BA-0053",
      "change_type": "PROBLEM_DETECTED",
      "previous_status": "ATIVO",
      "current_status": "DEGRADADA",
      "guardian": "CAPTAIN",
      "description": "Barreira degradada detectada pelo sistema"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 1000,
    "total_records": 2500,
    "total_pages": 3
  }
}
```

---

### Problemas Ativos

Lista barreiras com problemas atualmente identificados.

**GET** `/api/active-problems`

**Resposta:**
```json
{
  "active_problems": [
    {
      "barrier_id": "LGS-BA-0053",
      "site": "LGS",
      "equipment": "LGS-BA-0053",
      "status": "DEGRADADA",
      "description": "Barreira de segurança comprometida",
      "criticality": 3,
      "guardian_notified": "CAPTAIN",
      "notification_time": "2025-09-02T10:00:00Z",
      "hours_open": 7.2
    }
  ],
  "total_count": 11,
  "by_criticality": {
    "1": 2,
    "2": 3,
    "3": 4,
    "4": 2
  }
}
```

---

### Notificações

Histórico de notificações enviadas aos guardiões.

**GET** `/api/notifications`

**Parâmetros de Query:**

| Parâmetro | Tipo | Valores | Descrição |
|-----------|------|---------|-----------|
| `site` | string | ATS, GDS, LNS, etc. | Filtrar por site |
| `status` | string | SUCCESS, FAILED | Status da notificação |

**Exemplo:**
```
GET /api/notifications?site=LGS&status=SUCCESS
```

**Resposta:**
```json
{
  "data": [
    {
      "timestamp": "2025-09-02T10:00:00Z",
      "barrier_id": "LGS-BA-0053",
      "site": "LGS",
      "guardian": "CAPTAIN",
      "guardian_email": "captain@empresa.com",
      "status": "SUCCESS",
      "delivery_time_ms": 245
    }
  ],
  "total_notifications": 150,
  "success_rate": 0.98
}
```

---

### Respostas dos Guardiões

Respostas coletadas do sistema de votação dos guardiões.

**GET** `/api/guardian-responses`

**Parâmetros de Query:**

| Parâmetro | Tipo | Formato | Descrição |
|-----------|------|---------|-----------|
| `start_date` | string | YYYY-MM-DD | Data inicial |
| `end_date` | string | YYYY-MM-DD | Data final |

**Exemplo:**
```
GET /api/guardian-responses?start_date=2025-09-01&end_date=2025-09-02
```

**Resposta:**
```json
{
  "data": [
    {
      "timestamp": "2025-09-02T11:30:00Z",
      "barrier_id": "LGS-BA-0053",
      "guardian_email": "captain@empresa.com",
      "response": "Mitigar e Prosseguir",
      "justification": "Medidas mitigadoras implementadas"
    }
  ],
  "total_responses": 45
}
```

---

### Sites Operacionais

Lista todos os sites monitorados pelo sistema.

**GET** `/api/sites`

**Resposta:**
```json
{
  "sites": [
    "ADS", "AMS", "APS", "ATS", 
    "BVS", "GDS", "LGS", "LNS", "TAS"
  ],
  "total_sites": 9
}
```

---

## Modelos de Dados

### Barreira
```typescript
{
  barrier_id: string;      // Identificador único
  site: string;           // Código do site (3 letras)
  equipment: string;      // Equipamento associado
  status: string;         // ATIVO | DEGRADADA | INATIVA
  description: string;    // Descrição da barreira
  criticality: number;    // Nível crítico (1-4)
}
```

### Mudança de Status
```typescript
{
  timestamp: string;      // ISO 8601 format
  barrier_id: string;     // ID da barreira
  change_type: string;    // Tipo de mudança
  previous_status: string; // Status anterior
  current_status: string;  // Status atual
  guardian: string;       // Guardião responsável
}
```

### Notificação
```typescript
{
  timestamp: string;      // ISO 8601 format
  barrier_id: string;     // ID da barreira
  guardian: string;       // Tipo de guardião
  status: string;         // SUCCESS | FAILED
  delivery_time_ms: number; // Tempo de entrega
}
```

---

## Integração com PowerBI

### Configuração da Fonte de Dados

1. **Novo Dados Web**
   - Abrir PowerBI Desktop
   - Selecionar "Obter Dados" → "Web"
   - URL: `https://api.constellation-barriers.com/api/dashboard-summary`

2. **Configurar Headers**
   - Clicar em "Avançado"
   - Adicionar header: `X-API-Key` com sua chave

### Query M para Dashboard Summary

```m
let
    Source = Web.Contents(
        "https://api.constellation-barriers.com/api/dashboard-summary",
        [Headers=[#"X-API-Key"="sua-chave-aqui"]]
    ),
    JsonData = Json.Document(Source)
in
    JsonData
```

### Query M para Histórico de Mudanças

```m
let
    Source = Web.Contents(
        "https://api.constellation-barriers.com/api/change-history",
        [Headers=[#"X-API-Key"="sua-chave-aqui"]]
    ),
    JsonData = Json.Document(Source),
    DataList = JsonData[data],
    ToTable = Table.FromList(DataList, Splitter.SplitByNothing()),
    ExpandTable = Table.ExpandRecordColumn(
        ToTable, "Column1",
        {"timestamp", "barrier_id", "site", "change_type", "current_status"}
    )
in
    ExpandTable
```

### Configuração de Atualização

- **Frequência Recomendada:** 15 minutos
- **Método de Autenticação:** Anônimo (chave no header)
- **Nível de Privacidade:** Organizacional

---

## Códigos de Status

### Sucesso
| Código | Significado |
|--------|-------------|
| 200 | OK - Requisição bem-sucedida |

### Erros do Cliente
| Código | Significado | Descrição |
|--------|-------------|-----------|
| 400 | Bad Request | Parâmetros inválidos |
| 401 | Unauthorized | Chave de API inválida |
| 404 | Not Found | Endpoint não encontrado |
| 429 | Too Many Requests | Limite de requisições excedido |

### Erros do Servidor
| Código | Significado | Ação |
|--------|-------------|-------|
| 500 | Internal Server Error | Contatar suporte |
| 503 | Service Unavailable | Tentar novamente em alguns minutos |

---

## Limites de Uso

| Recurso | Limite |
|---------|--------|
| Requisições por minuto | 100 |
| Requisições por hora | 2000 |
| Registros por página | 5000 (máximo) |
| Timeout de requisição | 30 segundos |

---

## Formato de Resposta

Todas as respostas são em formato JSON com encoding UTF-8.

### Resposta de Sucesso
```json
{
  "data": [...],
  "timestamp": "2025-09-02T17:00:00Z"
}
```

### Resposta de Erro
```json
{
  "error": "Descrição do erro",
  "error_code": "INVALID_API_KEY",
  "timestamp": "2025-09-02T17:00:00Z"
}
```

---

## Exemplos de Implementação

### Python
```python
import requests

headers = {'X-API-Key': 'sua-chave-aqui'}
url = 'https://api.constellation-barriers.com/api/dashboard-summary'

response = requests.get(url, headers=headers)
data = response.json()

print(f"Total de barreiras: {data['summary']['total_barriers']}")
```

### JavaScript
```javascript
const headers = {'X-API-Key': 'sua-chave-aqui'};

fetch('https://api.constellation-barriers.com/api/dashboard-summary', {
    headers: headers
})
.then(response => response.json())
.then(data => {
    console.log('Total de barreiras:', data.summary.total_barriers);
});
```

### cURL
```bash
# Dashboard Summary
curl -H "X-API-Key: sua-chave-aqui" \
     https://api.constellation-barriers.com/api/dashboard-summary

# Problemas Ativos
curl -H "X-API-Key: sua-chave-aqui" \
     https://api.constellation-barriers.com/api/active-problems

# Histórico com Paginação
curl -H "X-API-Key: sua-chave-aqui" \
     "https://api.constellation-barriers.com/api/change-history?page=1&per_page=100"
```

---

## Suporte

Para dúvidas sobre a API ou solicitação de acesso:

- **Documentação:** https://docs.constellation-barriers.com
- **Suporte Técnico:** suporte@constellation-barriers.com
- **Status da API:** https://status.constellation-barriers.com

---

**Versão da API:** 1.0  
**Última Atualização:** Setembro 2025
