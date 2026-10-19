# Gustavo Borges · Database Engineering

Source of [guborges.github.io](https://guborges.github.io): technical articles on Oracle, PostgreSQL and MySQL, written from real production work.

```sql
SQL> SELECT DISTINCT topic, depth
  2  FROM   blog.posts
  3  WHERE  author = 'Gustavo Borges'
  4  ORDER  BY 1;

TOPIC                              DEPTH
---------------------------------- ------------
Exadata, RAC, ASM                  production
Data Guard, RMAN                   production
Performance (AWR, ASH, ADDM)       deep dive
PostgreSQL, MySQL                  hands-on
Oracle 26ai, AI + database         lab

5 rows selected.
```

## About

Oracle ACE Associate · OCP · AWS Certified. Database Administrator working with mission-critical Oracle environments, plus PostgreSQL and MySQL.

## Stack

Hugo with the PaperMod theme, built and deployed to GitHub Pages by GitHub Actions on every push to `main`.

## Elsewhere

[LinkedIn](https://www.linkedin.com/in/gustavoborges/) · [YouTube](https://www.youtube.com/@DBAGustavoBorges) · [RSS](https://guborges.github.io/index.xml)
