# Security Audit — 深度防御 7 层模型

Layer 1: Network (nginx rate limit + HTTPS)  
Layer 2: Auth (X-API-Key + HMAC signature)  
Layer 3: RBAC (4 roles + permission check)  
Layer 4: Guardrails (injection + PII + output)  
Layer 5: Circuit Breaker (fault isolation)  
Layer 6: Key Mgmt (AES-256-GCM encryption)  
Layer 7: Audit (full audit trail)
