---
title: "ORACLE – Data Guard Apply Lag Monitor"
date: 2025-12-29T07:00:00-03:00
draft: false
categories: ["Data Guard", "Oracle DBA", "Scripts & Automação"]
tags: ["Oracle", "Script Oracle"]
cover:
  image: "dglag.png"
---

Apply Lag zerado nem sempre significa que o Data Guard está saudável.  
Em ambientes RAC e de alta geração de redo, confiar em uma única métrica costuma mascarar problemas.

Neste artigo apresento um monitor simples de Apply Lag que cruza informações do Data Guard para dar **visibilidade real do estado de replicação**, focado em operação de produção.

Pontos abordados:

- Coleta de Apply Lag diretamente do banco
- Execução em múltiplos ambientes
- Saída padronizada para automação
- Base para alertas e monitoramento contínuo

O objetivo é **detectar problemas antes do impacto**, sem depender exclusivamente de OEM ou validações manuais.

🔗 <https://github.com/guborges/Data-Guard-Apply-Lag-Monitor>
