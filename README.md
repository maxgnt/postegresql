# PostgreSQL - Laboratoires

## Environnement

- Mac M2 8GB RAM
- Docker Desktop
- PostgreSQL 18.6
- DBeaver Community

---

## Lab 01 - Configuration de l'environnement

### Installation Docker + PostgreSQL 18

```bash
docker pull postgres:18
docker volume create pgdata
docker run -d --name postgres-dev \
  -e POSTGRES_PASSWORD=devpassword \
  -e POSTGRES_USER=devuser \
  -e POSTGRES_DB=blogapp \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql \
  postgres:18
```

Note : PostgreSQL 18 change le chemin PGDATA. Le volume se monte sur `/var/lib/postgresql` et non `/var/lib/postgresql/data` comme en PG 16.

### Structure de la table test_users

```sql
CREATE TABLE test_users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

```
   Column   |           Type           | Nullable |                Default
------------+--------------------------+----------+---------------------------------------------
 user_id    | integer                  | not null | nextval('test_users_user_id_seq'::regclass)
 username   | character varying(50)    | not null |
 email      | character varying(255)   | not null |
 created_at | timestamp with time zone |          | CURRENT_TIMESTAMP
```

### Configuration PostgreSQL

| Parametre            | Valeur |
| -------------------- | ------ |
| shared_buffers       | 128MB  |
| work_mem             | 4MB    |
| maintenance_work_mem | 64MB   |
| max_connections      | 100    |
| log_statement        | none   |

### Insertion et suppression de donnees

Insertion de 3 utilisateurs (alice, bob, charlie) puis suppression de charlie :

```sql
DELETE FROM test_users WHERE username = 'charlie';
```

```
 user_id | username |         email         |          created_at
---------+----------+-----------------------+-------------------------------
       2 | bob      | bob@example.com       | 2026-08-24 10:01:35.800229+00
       1 | alice    | alice.new@example.com | 2026-08-24 10:01:24.764322+00
```

### Captures

#### Insertion de charlie

![Insertion de charlie](captures/lab01/insert_user_charlie.png)

#### Visualisation dans DBeaver

![Charlie dans DBeaver](captures/lab01/user_charlie_dbeaver.png)

#### Suppression de charlie (DBeaver)

![Suppression charlie DBeaver](captures/lab01/remove_charlie_dbeaver.png)

---

### Commandes Docker utiles

```bash
docker stop postgres-dev      # Arreter
docker start postgres-dev     # Demarrer
docker restart postgres-dev   # Redemarrer
docker logs postgres-dev      # Voir les logs
docker exec -it postgres-dev psql -U devuser -d blogapp  # Se connecter
```

### Commandes psql essentielles

| Commande         | Description                 |
| ---------------- | --------------------------- |
| `\l`             | Lister les bases de donnees |
| `\dt`            | Lister les tables           |
| `\d nom_table`   | Decrire une table           |
| `\du`            | Lister les roles            |
| `\timing on/off` | Chronometrage des requetes  |
| `\x`             | Affichage etendu            |
| `\q`             | Quitter psql                |
