# AI Upsell, Cross-Sell & Next-Best-Action Engine (n8n): n8n Workflow Code

Importable n8n workflow (n8n Cloud). Copy the JSON below into a file named `next-best-action-aiagent-cloud.json`, then in n8n use **Workflows → Import from File** (or paste it onto the canvas with Ctrl+V).

After import, connect your own credentials (no secrets are stored in this file) and follow the setup steps in the README.

````json
{
  "name": "AI upsell, cross-sell & Next-Best-Action Engine (AI Agent, n8n Cloud)",
  "nodes": [
    {
      "parameters": {
        "content": "## Setup (n8n Cloud)\n1. Credentials (free tiers): Google Gemini, Slack, Postgres (Supabase)\n2. Create tables products (with seed rows) and recommendations in Supabase SQL Editor\n3. Slack channel: sales-opportunities (change in the Slack node)\n4. Click Execute workflow: 4 mock customers get a next-best-action each\nReal client use: replace Mock Customers with a Postgres select on your customer table + Schedule Trigger\nAI Agent node (Gemini) (no tools, JSON answer)",
        "height": 260,
        "width": 560
      },
      "id": "2c0e8b68-1594-4b64-9c8b-f531336b6b2e",
      "name": "Setup notes",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [
        -60,
        -220
      ]
    },
    {
      "parameters": {},
      "id": "5523e505-8626-488d-8bf8-74f0aef1a965",
      "name": "When clicking Execute workflow",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [
        0,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return [\n  { json: { customer: 'Northwind Traders', plan: 'Starter', usage_pct: 92, products_owned: 'CRM Starter', months: 14, open_tickets: 0 } },\n  { json: { customer: 'Bluebird Clinics', plan: 'Pro', usage_pct: 55, products_owned: 'CRM Pro, Email Add-on', months: 30, open_tickets: 1 } },\n  { json: { customer: 'Tailspin Logistics', plan: 'Pro', usage_pct: 40, products_owned: 'CRM Pro', months: 5, open_tickets: 6 } },\n  { json: { customer: 'Fabrikam Studio', plan: 'Enterprise', usage_pct: 88, products_owned: 'CRM Enterprise, Analytics Pack, Email Add-on', months: 40, open_tickets: 0 } }\n];"
      },
      "id": "5bfebfd1-7303-4a09-8b59-029c3305dbc9",
      "name": "Mock Customers",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        220,
        200
      ]
    },
    {
      "parameters": {
        "operation": "select",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "products",
          "mode": "name"
        },
        "options": {},
        "returnAll": true
      },
      "id": "84b975c3-390f-4c00-81fc-fb22b18c5773",
      "name": "Get Products",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        440,
        200
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      },
      "executeOnce": true
    },
    {
      "parameters": {
        "jsCode": "const catalog = $input.all().map((p) => p.json.name + ' ($' + p.json.price + '/mo, ' + p.json.category + '): ' + p.json.best_for).join('\\n');\nreturn $('Mock Customers').all().map((c) => ({ json: Object.assign({}, c.json, { catalog }) }));"
      },
      "id": "d581f6c0-5b4a-488f-9c9b-1d7ed4fe7d6d",
      "name": "Combine Catalog",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        660,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=You are a customer success strategist. Pick the single best next action for this customer.\nCustomer: {{ $json.customer }}\nPlan: {{ $json.plan }}\nUsage of plan limits: {{ $json.usage_pct }}%\nProducts owned: {{ $json.products_owned }}\nMonths as customer: {{ $json.months }}\nOpen support tickets: {{ $json.open_tickets }}\n\nProduct catalog:\n{{ $json.catalog }}\n\nChoose action: upsell (move to a higher plan), cross_sell (add a different product), retention (fix problems before selling) or none. Only recommend a product from the catalog. Give a confidence 0-100 and a one-sentence reason.",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a customer success strategist. Respond with ONLY a JSON object, no markdown, no extra text: {\"action\":\"upsell|cross_sell|retention|none\",\"product\":\"a product name from the catalog, or empty\",\"confidence\":0,\"reason\":\"one sentence\"}. confidence is a number from 0 to 100. Only recommend products that appear in the catalog."
        }
      },
      "id": "f9cb7bdc-88cd-466f-90d8-3717cfda3252",
      "name": "AI Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        880,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "modelName": "models/gemini-2.5-flash",
        "options": {}
      },
      "id": "332fa9fe-ec39-4d67-8cc8-b3024c239fbd",
      "name": "Google Gemini Chat Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        860,
        420
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "REPLACE_GEMINI",
          "name": "Google Gemini(PaLM) Api account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Combine Catalog').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst action = ['upsell', 'cross_sell', 'retention', 'none'].includes(r.action) ? r.action : 'none';\nconst confidence = Number(r.confidence) || 0;\nreturn { json: {\n  customer: ctx.customer,\n  plan: ctx.plan,\n  action,\n  product: r.product || '',\n  confidence,\n  reason: String(r.reason || '').slice(0, 400),\n  status: (action !== 'none' && confidence >= 70) ? 'strong' : 'weak'\n}};"
      },
      "id": "b0ffe9ea-d6ea-491e-a62c-f600f3568251",
      "name": "Parse Recommendation",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1240,
        200
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "9b3538ef-b281-4164-9c37-88ac7861b7f3",
              "leftValue": "={{ $json.status }}",
              "rightValue": "strong",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "18002c9d-12af-4b8f-ab71-a93cda3bae1c",
      "name": "Strong Opportunity?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        1460,
        200
      ]
    },
    {
      "parameters": {
        "select": "channel",
        "channelId": {
          "__rl": true,
          "value": "sales-opportunities",
          "mode": "name"
        },
        "text": "=:dart: *Next best action: {{ $json.action }}*\nCustomer: {{ $json.customer }} ({{ $json.plan }})\nProduct: {{ $json.product }}\nConfidence: {{ $json.confidence }}%\nWhy: {{ $json.reason }}",
        "otherOptions": {}
      },
      "id": "e2600898-a8d1-4812-9868-f62dcceca632",
      "name": "Slack Alert",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2.3,
      "position": [
        1720,
        120
      ],
      "credentials": {
        "slackApi": {
          "id": "REPLACE_SLACK",
          "name": "Slack account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "operation": "insert",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "recommendations",
          "mode": "name"
        },
        "options": {},
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "customer": "={{ $json.customer }}",
            "plan": "={{ $json.plan }}",
            "action": "={{ $json.action }}",
            "product": "={{ $json.product }}",
            "confidence": "={{ $json.confidence }}",
            "reason": "={{ $json.reason }}",
            "status": "={{ $json.status }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "customer",
              "displayName": "customer",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "plan",
              "displayName": "plan",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "action",
              "displayName": "action",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "product",
              "displayName": "product",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "confidence",
              "displayName": "confidence",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "reason",
              "displayName": "reason",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "status",
              "displayName": "status",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            }
          ]
        }
      },
      "id": "52027810-5721-4c9c-9580-814ee4cfce54",
      "name": "Save Recommendation",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        1720,
        320
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      }
    }
  ],
  "connections": {
    "When clicking Execute workflow": {
      "main": [
        [
          {
            "node": "Mock Customers",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Mock Customers": {
      "main": [
        [
          {
            "node": "Get Products",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Get Products": {
      "main": [
        [
          {
            "node": "Combine Catalog",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Combine Catalog": {
      "main": [
        [
          {
            "node": "AI Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Recommendation": {
      "main": [
        [
          {
            "node": "Strong Opportunity?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Strong Opportunity?": {
      "main": [
        [
          {
            "node": "Slack Alert",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Recommendation",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Save Recommendation",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Gemini Chat Model": {
      "ai_languageModel": [
        [
          {
            "node": "AI Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "AI Agent": {
      "main": [
        [
          {
            "node": "Parse Recommendation",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "tags": []
}
````
