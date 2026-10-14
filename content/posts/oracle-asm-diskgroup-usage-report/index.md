---
title: "ORACLE – ASM Diskgroup Usage Report"
date: 2026-01-05T07:00:00-03:00
draft: false
categories: ["ASM & Storage", "Oracle DBA", "Scripts & Automação"]
tags: ["Oracle", "Script Oracle"]
cover:
  image: "asmenchendo.png"
---

Monitorar ASM apenas olhando espaço livre costuma gerar uma falsa sensação de segurança.  
Em ambientes de produção, o que importa é entender **uso real, espaço disponível e tendência**.

Este projeto gera um relatório simples de diskgroups ASM, com foco em operação diária:

- Total, usado e livre por diskgroup
- Percentual de utilização
- Saída clara para uso em:
  - cron
  - e-mail
  - monitoramento

O objetivo é ter **visibilidade rápida** sem depender de OEM ou consultas manuais.

🔗 <https://github.com/guborges/oracle-asm-diskgroup-usage-report>
