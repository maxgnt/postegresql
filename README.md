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

### Informations de connexion

```
 Parameter             | Value
-----------------------+---------------------
 Database              | blogapp
 Client User           | devuser
 Socket Directory      | /var/run/postgresql
 Server Port           | 5432
 Protocol Version      | 3.0
 Superuser             | on
```

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

### Transactions

Transaction avec COMMIT (dave est conserve) :

```
blogapp=# BEGIN;
BEGIN
blogapp=*# INSERT INTO test_users (username, email) VALUES ('dave', 'dave@example.com');
INSERT 0 1
blogapp=*# COMMIT;
COMMIT
blogapp=# SELECT * FROM test_users;
 user_id | username |         email         |          created_at
---------+----------+-----------------------+-------------------------------
       2 | bob      | bob@example.com       | 2026-08-24 10:01:35.800229+00
       1 | alice    | alice.new@example.com | 2026-08-24 10:01:24.764322+00
       5 | dave     | dave@example.com      | 2026-08-24 10:29:25.37553+00
(3 rows)
```

Transaction avec ROLLBACK (eve est annulee) :

```
blogapp=# BEGIN;
BEGIN
blogapp=*# INSERT INTO test_users (username, email) VALUES ('eve', 'eve@example.com');
INSERT 0 1
blogapp=*# SELECT * FROM test_users;
 user_id | username |         email         |          created_at
---------+----------+-----------------------+-------------------------------
       2 | bob      | bob@example.com       | 2026-08-24 10:01:35.800229+00
       1 | alice    | alice.new@example.com | 2026-08-24 10:01:24.764322+00
       5 | dave     | dave@example.com      | 2026-08-24 10:29:25.37553+00
       6 | eve      | eve@example.com       | 2026-08-24 10:33:02.298717+00
(4 rows)

blogapp=*# ROLLBACK;
ROLLBACK
blogapp=# SELECT * FROM test_users;
 user_id | username |         email         |          created_at
---------+----------+-----------------------+-------------------------------
       2 | bob      | bob@example.com       | 2026-08-24 10:01:35.800229+00
       1 | alice    | alice.new@example.com | 2026-08-24 10:01:24.764322+00
       5 | dave     | dave@example.com      | 2026-08-24 10:29:25.37553+00
(3 rows)
```

Le ROLLBACK a annule l'insertion d'eve. Seuls alice, bob et dave restent.

### Captures

#### Connexion DBeaver

![Connexion DBeaver](captures/lab01/connexion_dbeaver.png)

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
| `\conninfo`      | Informations de connexion   |
| `\timing on/off` | Chronometrage des requetes  |
| `\x`             | Affichage etendu            |
| `\q`             | Quitter psql                |

---

## Lab 02 - DDL & DML : Conception de schema et manipulation de donnees

### Chargement de la base Pagila

Pagila est la base d'exemple officielle PostgreSQL (magasin de location de DVD). Chargement dans le conteneur :

```bash
docker exec -it postgres-dev bash
apt update && apt install -y wget
wget https://raw.githubusercontent.com/devrimgunduz/pagila/master/pagila-schema.sql
wget https://raw.githubusercontent.com/devrimgunduz/pagila/master/pagila-data.sql
psql -U devuser -d postgres -c "CREATE DATABASE pagila;"
psql -U devuser -d pagila -f pagila-schema.sql
psql -U devuser -d pagila -f pagila-data.sql
```

Erreurs normales lors du chargement :

- "role postgres does not exist" : sans impact, les objets sont crees avec devuser
- "film_embedding" / "vector" : extension pgvector non installee, pas necessaire pour le TP

### Exercice 2.1 : Exploration du schema Pagila

Tables principales : actor, address, category, city, country, customer, film, film_actor, film_category, inventory, language, payment (partitionnee), rental, staff, store.

Structure de la table customer :

```
   Column    |           Type           | Nullable |                    Default
-------------+--------------------------+----------+-----------------------------------------------
 customer_id | integer                  | not null | nextval('customer_customer_id_seq'::regclass)
 store_id    | integer                  | not null |
 first_name  | text                     | not null |
 last_name   | text                     | not null |
 email       | text                     |          |
 address_id  | integer                  | not null |
 activebool  | boolean                  | not null | true
 create_date | date                     | not null | CURRENT_DATE
 last_update | timestamp with time zone |          | now()
 active      | integer                  |          |
 uuid        | uuid                     | not null | uuidv7()
```

Differences avec le TP du prof :

- `text` au lieu de `varchar(45)` pour first_name/last_name
- `integer` au lieu de `smallint` pour store_id/address_id
- Colonne `uuid` avec `uuidv7()` (nouveaute PostgreSQL 18)

### Exercice 2.2 : Analyse des relations (cles etrangeres)

```
   table_enfant   |    colonne_enfant    | table_parent | colonne_parent | update_rule | delete_rule
------------------+----------------------+--------------+----------------+-------------+-------------
 address          | city_id              | city         | city_id        | CASCADE     | RESTRICT
 city             | country_id           | country      | country_id     | CASCADE     | RESTRICT
 customer         | address_id           | address      | address_id     | CASCADE     | RESTRICT
 customer         | store_id             | store        | store_id       | CASCADE     | RESTRICT
 film             | language_id          | language     | language_id    | CASCADE     | RESTRICT
 film             | original_language_id | language     | language_id    | CASCADE     | RESTRICT
 film_actor       | actor_id             | actor        | actor_id       | CASCADE     | RESTRICT
 film_actor       | film_id              | film         | film_id        | CASCADE     | RESTRICT
 film_category    | category_id          | category     | category_id    | CASCADE     | RESTRICT
 film_category    | film_id              | film         | film_id        | CASCADE     | RESTRICT
 inventory        | film_id              | film         | film_id        | CASCADE     | RESTRICT
 inventory        | store_id             | store        | store_id       | CASCADE     | RESTRICT
 rental           | customer_id          | customer     | customer_id    | CASCADE     | RESTRICT
 rental           | inventory_id         | inventory    | inventory_id   | CASCADE     | RESTRICT
 rental           | staff_id             | staff        | staff_id       | CASCADE     | RESTRICT
 staff            | address_id           | address      | address_id     | CASCADE     | RESTRICT
 store            | address_id           | address      | address_id     | CASCADE     | RESTRICT
```

Les tables de payment partitionnees utilisent NO ACTION (similaire a RESTRICT).

Regles principales :

- CASCADE sur UPDATE : si l'ID parent change, les enfants suivent
- RESTRICT sur DELETE : impossible de supprimer un parent tant que des enfants existent
