# Shared Contacts

## Architecture

(REST API) (JWT Authorization)
(Websockets)

(Postgres/Mysql) (Memcache/Redis)

## Schema

```
CREATE TABLE users (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  first_name VARCHAR(50),
  last_name VARCHAR(50),
  pass_hash VARCHAR(100),
  role_id BIGINT,
  last_login TIMESTAMP,
  created_on TIMESTAMP,
  updated_on TIMESTAMP,
  delete_on TIMESTAMP,

  FOREIGN KEY (role_id)
    REFERENCES roles (id)
      ON UPDATE RESTRICT ON DELETE CASCADE
);

CREATE TABLE roles (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(50),
  created_on TIMESTAMP,
  updated_on TIMESTAMP,
  delete_on TIMESTAMP,
);

CREATE TABLE permissions (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  resource VARCHAR(50),
  action VARCHAR(50),
  contraints JSON(B),
  created_on TIMESTAMP,
  updated_on TIMESTAMP,
  delete_on TIMESTAMP,
);

// E.g. ('contact', 'view')
// E.g. ('contact', 'add')
// E.g. ('contact', 'edit')
// E.g. ('contact', 'edit', { field: ['address'] })


CREATE TABLE role_permissions (
  role_id BIGINT,
  permission_id BIGINT,

  FOREIGN KEY (role_id)
    REFERENCES roles (id)
      ON UPDATE RESTRICT ON DELETE CASCADE

  FOREIGN KEY (permission_id)
    REFERENCES permissions (id)
      ON UPDATE RESTRICT ON DELETE CASCADE
);

CREATE TABLE contact_lists (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(50),
  created_on TIMESTAMP,
  updated_on TIMESTAMP,
  delete_on TIMESTAMP,
);

CREATE TABLE contacts (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  first_name VARCHAR(50),
  last_name VARCHAR(50),
  email VARCHAR(50),
  title VARCHAR(50),
  company_id BIGINT,
  notes TEXT,
  search tsvector,
  last_verified_on TIMESTAMP,
  last_attemped_verification TIMESTAMP,
  total_verification_attempts TINYINT,
  created_on TIMESTAMP,
  updated_on TIMESTAMP,
  delete_on TIMESTAMP,

  FOREIGN KEY (company_id)
    REFERENCES companies (id)
      ON UPDATE RESTRICT ON DELETE CASCADE
  
  FOREIGN KEY (address_id)
    REFERENCES addresses (id)
      ON UPDATE RESTRICT ON DELETE CASCADE
  
  FOREIGN KEY (phone_number_id)
    REFERENCES phone_numbers (id)
      ON UPDATE RESTRICT ON DELETE CASCADE
);

// TODO - Add revision history

CREATE TABLE companies (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  created_on TIMESTAMP,
  updated_on TIMESTAMP,
  delete_on TIMESTAMP,
);

CREATE TABLE addresses (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  address_type_id BIGINT,
  address1 VARCHAR(50),
  address1 VARCHAR(50),
  city VARCHAR(50),
  state VARCHAR(5),
  postal_code VARCHAR(15),
  created_on TIMESTAMP,
  updated_on TIMESTAMP,
  delete_on TIMESTAMP,

  FOREIGN KEY (address_type_id)
    REFERENCES address_types (id)
      ON UPDATE RESTRICT ON DELETE CASCADE
);

CREATE TABLE address_types (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  type VARCHAR(50),
  created_on TIMESTAMP,
  updated_on TIMESTAMP,
  delete_on TIMESTAMP,
);

CREATE TABLE user_addresses (
  user_id BIGINT,
  address_id BIGINT,

  FOREIGN KEY (user_id)
    REFERENCES users (id)
      ON UPDATE RESTRICT ON DELETE CASCADE

  FOREIGN KEY (address_id)
    REFERENCES addresses (id)
      ON UPDATE RESTRICT ON DELETE CASCADE
);

CREATE TABLE phone_number_types (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  type VARCHAR(50),
  created_on TIMESTAMP,
  updated_on TIMESTAMP,
  delete_on TIMESTAMP,
);

CREATE TABLE phone_numbers (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  phone_number_type_id BIGINT,
  number VARCHAR(100),
  created_on TIMESTAMP,
  updated_on TIMESTAMP,
  delete_on TIMESTAMP,

  FOREIGN KEY (phone_number_type_id)
    REFERENCES phone_number_types (id)
      ON UPDATE RESTRICT ON DELETE CASCADE
);

CREATE TABLE user_phone_numbers (
  user_id BIGINT,
  phone_number_id BIGINT,

  FOREIGN KEY (user_id)
    REFERENCES users (id)
      ON UPDATE RESTRICT ON DELETE CASCADE

  FOREIGN KEY (phone_number_id)
    REFERENCES phone_numbers (id)
      ON UPDATE RESTRICT ON DELETE CASCADE
);

CREATE TABLE user_contact_list (
  user_id BIGINT,
  contact_list_id BIGINT,

  FOREIGN KEY (user_id)
    REFERENCES users (id)
      ON UPDATE RESTRICT ON DELETE CASCADE
  
  FOREIGN KEY (contact_list_id)
    REFERENCES contact_list (id)
      ON UPDATE RESTRICT ON DELETE CASCADE
);
```

## Verification

Cloudwatch event setup to kick off a Lambda, run once a day. Find contacts that haven't been verified in over 90(?) days. Send an email message with a link which has a link to a verification link and a AES-256-CBC encrypted value which is their email address and a server-side secret/IV. The `last_attempted_verification_on` and `total_attempted_verifications` would be updated for that contact record. The verification link calls an API endpoint which decrypts and then updates the contact entry field for `last_verified_on` and reset `last_attempted_verification_on` and `total_attempted_verifications`.

## Concurrency

- Choosing not to go the CDRT route because it is beyond annoying in real-time collaborative tools to have your content changed without locking controls.

- Field and object level locking on contacts will exist.
  
  * Using WATCH with redis, contacts/fields are locked when edit is requested from the user. A websocket event is broadcast to all sessions and that object/field is locked until the lock is removed.

  * Redis record: [contact id]: [user id] or [contact id/field]: [user id], will have a default TTL of 5mins on it so a user cannot lock the record/field forever. The frontend will be responsible for sending socket messages if the field blurs or user clicks Cancel/Save on the record or moves to another route to release the lock before the TTL.

  * Could leverage a queue to let user request the lock even while another user has it, so a fair usage policy is in play.

- Updates all done via transactions to the database record.

## API

```
GET /user (returns current authenticated user info)
POST /user (user registration?)
PUT /user (user profile update)
GET /user/contact-lists (returns current authenticated users contact lists)

GET /contact-list/:id? (contact list info)
GET /contact-list/:id/contacts (contacts associated to a contact list)

GET /contact/:id (specific contact)
POST /contact
PUT /contact/:id
DELETE /contact/:id
```

## Middleware

- User authenticated done via JWT
- User permissions/authorization
    ```
    const { can, rules } = AbilityBuilder.extract();
    user.roles.forEach(role => {
      role.permissions.forEach(permission => {
        can(permission.action, permission.resource);
      });
    });

    user.rbac = new Ability(rules);
    ```
  
  - middleware per route such as, ```rbac.authorize(['view','contact'])```

  - constraints processed at the validation level of the request
    - i.e. if user can only edit the contacts first and last name the validation 

## GDPR & CCPA Compliance

Cloudwatch event runs once daily, any contact records soft deleted within the 14 days or later are purged physically from the database.

## Mockups

[Login](https://raw.githubusercontent.com/Svjard/contacts/master/.github/Login.png)
[Lists](https://raw.githubusercontent.com/Svjard/contacts/master/.github/Lists.png)
[Search](https://raw.githubusercontent.com/Svjard/contacts/master/.github/Search.png)
[Edit](https://raw.githubusercontent.com/Svjard/contacts/master/.github/Edit.png)