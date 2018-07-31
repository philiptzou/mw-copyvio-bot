## Archtecture and Main Dependencies

- **Python** - backend server and scripts
  - **Flask** - web framework
  - **SQLAlchemy** - ORM
  - **Alembic** - DB migration
  - **Click** - command line framework
- **JavaScript** - frontend script running on a MediaWiki site
  - **React** - framework
- **PostgreSQL** - relationship database
- **Crontab** - scheduled tasks
- **Nginx** - web server
- **Docker** - backend container


## Configuration Design

### DB connection

See DB parameters of `flask-sqlalchemy`

### Site definition

| column                | type   | attributes                  | comment                                                                              |
|-----------------------|--------|-----------------------------|--------------------------------------------------------------------------------------|
| `site_name`           | `str`  | not null, unique            | Site name                                                                            |
| `domain_name`         | `str`  | not null                    | Domain name of the site, for example: `"en.wikipedia.org"`                           |
| `contlang`            | `str`  | not null                    | Content language of this wiki                                                        |
| `mw_path`             | `str`  | not null, default `'/w'`    | Location of MediaWiki software                                                       |
| `wiki_path`           | `str`  | not null, default `'/wiki'` | Location of MediaWiki "shortcut" path                                                |
| `report_page`         | `str`  | not null                    | CopyVio reporting Page name (include namespace); template format is allowed          |
| `report_page_section` | `str`  | not null                    | Section of the CopyVio reporting page; template format is allowed                    |
| `report_tpl`          | `str`  | not null, default (long)    | Template for report a CopyVio page for deletion                                      |
| `revdel_page`         | `str`  |                             | CopyVio revision reporting Page name (include namespace); template format is allowed |
| `revdel_page_section` | `str`  |                             | Section of the CopyVio revision reporting page; template format is allowed           |
| `revdel_tpl`          | `str`  | not null, default (long)    | Template for report a range of CopyVio revisions for deletion                        |
| `notify_user`         | `bool` | not null                    | Whether to notify the involved user or not on this wiki                              |
| `notify_user_tpl`     | `str`  |                             | Template for notifying user (if the `notify_user` is on)                             |


## Database Design

### Table `pages`

A table stores meta information of MediaWiki pages per sites

| column              | type                                                   | attributes            | comment                                    |
|---------------------|--------------------------------------------------------|-----------------------|--------------------------------------------|
| `id`                | `integer`                                              | primary key, not null | Table primary key                          |
| `site_name`         | `unicode(64)`                                          | not null              | Site name from config                      |
| `site_page_id`      | `integer`                                              | not null, index       | Page ID on the wiki                        |
| `title`             | `unicode(1024)`                                        | not null              | Page title without namespace               |
| `ns_title`          | `unicode(64)`                                          | index, default `null` | Namespace title; `null` means main space   |
| `status`            | `enum(OK, REVIEWING, CONFIRMED, ROLLED_BACK, DELETED)` | not null, index       | Page status:<br />REVIEWING: possible copyvio detected; awaiting for review<br />CONFIRMED: a human confirmed copyvio; awaiting for final process<br />ROLLED\_BACK: a human confirmed the copyvio and rolled back to a previous revision; awaiting for revision deletion<br />DELETED: page has been deleted<br />OK: no copyvio issue at current revision |
| `datetime_modified` | `datetime(UTC)`                                        | index                 | The last datetime this report was modified |

### Table `users`

| column         | type           | attributes            | comment               |
|----------------|----------------|-----------------------|-----------------------|
| `id`           | `integer`      | primary key, not null | Table primary key     |
| `site_name`    | `unicode(64)`  | not null              | Site name from config |
| `site_user_id` | `integer`      | not null, index       | User ID on the wiki   |
| `user_name`    | `unicode(256)` | not null, index       | User name             |

### Table `reports`

A table stores copyright violation reports

| column                  | type                             | attributes                        | comment                                                         |
|-------------------------|----------------------------------|-----------------------------------|-----------------------------------------------------------------|
| `id`                    | `integer`                        | primary key, not null             | Table primary key                                               |
| `page_id`               | `integer`                        | foreign key(`pages.id`), not null | Foreign key to `pages` table                                    |
| `copyvio_revids`        | `unicodeText`                    | not null                          | A list of involved revision ID in the MediaWiki site            |
| `latest_revid`          | `integer`                        | not null                          | The latest status changing revision ID in this report           |
| `rollback_option`       | `bool`                           | not null, index                   | If a previous revision (for rolling back) is available          |
| `rollback_target_revid` | `integer`                        |                                   | A previous revision for rolling back                            |
| `detail`                | `unicodeText`                    | not null                          | JSON structured detailed report, stores diff and involved users |
| `status`                | `enum(NEW, CONFIRMED, REJECTED)` | not null, index                   | Report status:<br />NEW: possible copyvio deteced, awaiting for review<br />CONFIRMED: a human confirmed this report<br />REJECTED: a human rejected this report |
| `datetime_created`      | `datetime(UTC)`                  | not null, index                   | The datetime this report first created                          |
| `datetime_modified`     | `datetime(UTC)`                  | index                             | The last datetime this report was modified                      |

### Table `report_involved_users`

| column         | type           | attributes                                       | comment               |
|----------------|----------------|--------------------------------------------------|-----------------------|
| `report_id`    | `integer`      | primary key, foreign key(`reports.id`), not null | One of primary keys   |
| `user_id`      | `integer`      | primary key, foreign key('users.id`), not null   | One of primary keys   |

### Table `report_logs`

| column      | type            | attributes                         | comment                        |
|-------------|-----------------|------------------------------------|--------------------------------|
| `id`        | `integer`       | primary key, not null              | Table primary key              |
| `report_id` | `integer`       | foreign key(`report.id`), not null | Foreign key to `reports` table |
| `user_id`   | `unicode(1024)` | foreign key(`user.id`), not null   | Foreign key to `users` table   |
| `operation` | `enum()`        | not null, index                    | Operation this user did (TODO) |
| `detail`    | `unicodeText`   | not null                           | JSON structured detail         |
| `datetime`  | `datetime(UTC)` | not null, index                    | Datetime of this operation     |

### Table `whitelist`

Whitelist for websites exclued from copyvio checking

| column        | type            | attributes            | comment                 |
|---------------|-----------------|-----------------------|-------------------------|
| `id`          | `integer`       | primary key, not null | Table primary key       |
| `url_pattern` | `unicodeText`   | not null              | Regular expression      |
| `reason`      | `unicode(64)`   |                       | License or other reason |


## API Design

### `SearchEngine` class

A class that defined the general interface to any SearchEngine.

| property/method     | description                                  |
|---------------------|----------------------------------------------|
| `query(q, dt_end)`  | Search for a given query                     |
| `usage_info`        | Structured data of daily usage information   |

This interface can be used to implement Google Search, DuckDuckGo, Bing, Google Book Search, etc.

### `Content` class

A class that defined the general interface for processing content.


| property/method         | description                                                    |
|-------------------------|----------------------------------------------------------------|
| `get_text()`            | Convert rich-format text to plain text                         |
| `compare(another)`      | Compare with another `Content` instance and output differences |
| `usage_info`            | Structured data of daily usage information                     |

`MWContent`, `HTMLContent` can be implemented as subclass of `Content`.



