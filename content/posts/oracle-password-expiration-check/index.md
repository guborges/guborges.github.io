---
title: "ORACLE – Password Expiration Check"
date: 2025-12-15T07:00:00-03:00
draft: false
categories: ["Oracle DBA", "Scripts & Automação", "Segurança"]
tags: ["Oracle", "Script Oracle"]
cover:
  image: "passex.png"
---

Em ambientes Oracle, a expiração de senha de usuários de banco não gera alerta nativo confiável e, na prática, costuma ser descoberta apenas após impacto em aplicação ou jobs.

Este projeto resolve esse ponto com um check simples e objetivo, focado em operação de produção.

### O que o script faz

- Identifica usuários com **senha expirada**
- Identifica usuários **próximos da expiração**
- Considera PROFILE, PASSWORD_LIFE_TIME e ACCOUNT_STATUS
- Saída padronizada, pronta para:
  - cron
  - e-mail
  - integração com monitoramento

### Por que isso é necessário

- Contas técnicas não fazem login interativo
- Alertas de expiração não são tratados automaticamente
- O erro aparece **somente no runtime da aplicação**

### Projeto

🔗 <https://github.com/guborges/oracle-password-expiration-check>

Projeto focado em **prevenção de incidente** e **estabilidade operacional**.
