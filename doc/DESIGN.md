## Archtecture and Main Dependencies

- **Python** - backend server and scripts
  - **Flask** - web framework
  - **SQLAlchemy** - ORM
  - **Click** - command line framework
- **JavaScript** - frontend script running on a MediaWiki site
- **PostgreSQL** - relationship database
- **Nginx** - web server
- **Docker** - backend container

## Database Design

### Table `sites`

A table stores meta information of MediaWiki sites (move to configuration?)

| column                | type            | attributes                  | comment                                                                              |
|-----------------------|-----------------|-----------------------------|--------------------------------------------------------------------------------------|
| `id`                  | `integer`       | primary key, not null       | Table primary key                                                                    |
| `site_name`           | `unicode(64)`   | not null, unique index      | Site name                                                                            |
| `domain_name`         | `unicode(128)`  | not null                    | Domain name of the site, for example: `"en.wikipedia.org"`                           |
| `mw_path`             | `unicode(64)`   | not null, default `'/w'`    | Location of MediaWiki software                                                       |
| `wiki_path`           | `unicode(64)`   | not null, default `'/wiki'` | Location of MediaWiki "shortcut" path                                                |
| `report_page`         | `unicode(1024)` | not null                    | CopyVio reporting Page name (include namespace); template format is allowed          |
| `report_page_section` | `unicode(1024)` | not null                    | Section of the CopyVio reporting page; template format is allowed                    |
| `report_tpl`          | `unicodeText`   | not null, default (long)    | Template for report a CopyVio page for deletion                                      |
| `revdel_page`         | `unicode(1024)` |                             | CopyVio revision reporting Page name (include namespace); template format is allowed |
| `revdel_page_section` | `unicode(1024)` |                             | Section of the CopyVio revision reporting page; template format is allowed           |
| `revdel_tpl`          | `unicodeText`   | not null, default (long)    | Template for report a range of CopyVio revisions for deletion                        |
| `notify_user`         | `bool`          | not null                    | Whether to notify the involved user or not on this wiki                              |
| `notify_user_tpl`     | `unicodeText`   |                             | Template for notifying user (if the `notify_user` is on)                             |


### Table `pages`

A table stores meta information of MediaWiki pages per sites

| column              | type                                                   | attributes                        | comment                                    |
|---------------------|--------------------------------------------------------|-----------------------------------|--------------------------------------------|
| `id`                | `integer`                                              | primary key, not null             | Table primary key                          |
| `site_id`           | `integer`                                              | foreign key(`sites.id`), not null | Foreign key to `sites` table               |
| `site_page_id`      | `integer`                                              | not null, index                   | Page ID on the wiki                        |
| `title`             | `unicode(1024)`                                        | not null                          | Page title without namespace               |
| `ns_title`          | `unicode(64)`                                          | index, default `null`             | Namespace title; `null` means main space   |
| `status`            | `enum(OK, REVIEWING, CONFIRMED, ROLLED_BACK, DELETED)` | not null, index                   | Page status:<br />REVIEWING: possible copyvio detected; awaiting for review<br />CONFIRMED: a human confirmed copyvio; awaiting for final process<br />ROLLED\_BACK: a human confirmed the copyvio and rolled back to a previous revision; awaiting for revision deletion<br />DELETED: page has been deleted<br />OK: no copyvio issue at current revision |
| `datetime_modified` | `datetime(UTC)`                                        | index                             | The last datetime this report was modified |

### Table `users`

| column         | type           | attributes                        | comment                      |
|----------------|----------------|-----------------------------------|------------------------------|
| `id`           | `integer`      | primary key, not null             | Table primary key            |
| `site_id`      | `integer`      | foreign key(`sites.id`), not null | Foreign key to `sites` table |
| `site_user_id` | `integer`      | not null, index                   | User ID on the wiki          |
| `user_name`    | `unicode(256)` | not null, index                   | User name                    |

### Table `reports`

A table stores copyright violation reports

| column              | type                             | attributes                        | comment                                                |
|---------------------|----------------------------------|-----------------------------------|--------------------------------------------------------|
| `id`                | `integer`                        | primary key, not null             | Table primary key                                      |
| `page_id`           | `integer`                        | foreign key(`pages.id`), not null | Foreign key to `pages` table                           |
| `site_revids`       | `unicodeText`                    | not null                          | A list of involved revision ID in the MediaWiki site   |
| `rollback_option`   | `bool`                           | not null, index                   | If a previous revision (for rolling back) is available |
| `rbtgt_site_revid`  | `integer`                        |                                   | A previous revision for rolling back                   |
| `involved_user_ids` | `unicodeText`                    | not null                          | Involved user ids (move to a many-to-many table?)      |
| `detail`            | `unicodeText`                    | not null                          | JSON structured detailed report                        |
| `status`            | `enum(NEW, CONFIRMED, REJECTED)` | not null, index                   | Report status:<br />NEW: possible copyvio deteced, awaiting for review<br />CONFIRMED: a human confirmed this report<br />REJECTED: a human rejected this report |
| `datetime_created`  | `datetime(UTC)`                  | not null, index                   | The datetime this report first created                 |
| `datetime_modified` | `datetime(UTC)`                  | index                             | The last datetime this report was modified             |

### Table `report_logs`

| column      | type            | attributes                         | comment                        |
|-------------|-----------------|------------------------------------|--------------------------------|
| `id`        | `integer`       | primary key, not null              | Table primary key              |
| `report_id` | `integer`       | foreign key(`report.id`), not null | Foreign key to `reports` table |
| `user_id`   | `unicode(1024)` | foreign key(`user.id`), not null   | Foreign key to `users` table   |
| `operation` | `enum()`        | not null, index                    | Operation this user did (TODO) |
| `detail`    | `unicodeText`   | not null                           | JSON structured detail         |
| `datetime`  | `datetime(UTC)` | not null, index                    | Datetime of this operation     |


## API Design
