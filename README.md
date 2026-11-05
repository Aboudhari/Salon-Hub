# Salon-Hub
SalonHub Rwanda is a multi-business salon, barbershop, nail studio, and beauty-service management platform combined with a public marketplace. Each registered business receives a private workspace for daily operations and a public profile that customers can discover through the shared marketplace.
 DATABASE
   id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT UNSIGNED NOT NULL,
  business_id BIGINT UNSIGNED NULL,
  channel ENUM('in_app','email','sms') NOT NULL DEFAULT 'in_app',
  title VARCHAR(180) NOT NULL,
  message TEXT NOT NULL,
  related_type VARCHAR(60),
  related_id BIGINT UNSIGNED NULL,
  status ENUM('pending','sent','failed','read') NOT NULL DEFAULT 'pending',
  sent_at DATETIME NULL,
  read_at DATETIME NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_notifications_user (user_id, status, created_at),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (business_id) REFERENCES businesses(id) ON DELETE SET NULL
) ENGINE=InnoDB;

CREATE TABLE platform_settings (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  setting_key VARCHAR(120) NOT NULL UNIQUE,
  setting_value TEXT,
  description VARCHAR(500),
  updated_by BIGINT UNSIGNED NULL,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (updated_by) REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB;

CREATE TABLE audit_logs (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  business_id BIGINT UNSIGNED NULL,
  user_id BIGINT UNSIGNED NULL,
  action VARCHAR(120) NOT NULL,
  entity_type VARCHAR(100) NOT NULL,
  entity_id BIGINT UNSIGNED NULL,
  old_values JSON NULL,
  new_values JSON NULL,
  ip_address VARCHAR(45),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_audit_business_date (business_id, created_at),
  INDEX idx_audit_user_date (user_id, created_at),
  FOREIGN KEY (business_id) REFERENCES businesses(id) ON DELETE SET NULL,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB;

-- Seed standard platform/business roles. Business-specific authorization
-- must additionally validate the user's active membership in the target business.
INSERT INTO roles (name, description) VALUES
 ('platform_admin','Platform-wide administrator'),
 ('business_owner','Business owner'),
 ('business_manager','Business manager'),
 ('receptionist','Front desk and appointment operations'),
 ('professional','Service professional'),
 ('customer','Marketplace customer');

INSERT INTO permissions (permission_key, description) VALUES
 ('platform.manage','Manage platform configuration and users'),
 ('business.manage','Manage business profile and settings'),
 ('staff.manage','Manage business staff'),
 ('services.manage','Manage services and prices'),
 ('appointments.manage','Manage appointments and queue'),
 ('customers.manage','Manage customer records'),
 ('sales.manage','Manage sales and payments'),
 ('expenses.manage','Manage expenses'),
 ('inventory.manage','Manage inventory and suppliers'),
 ('reports.view','View business reports'),
 ('marketplace.manage','Manage public business content');

INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r CROSS JOIN permissions p
WHERE r.name = 'platform_admin';

INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r JOIN permissions p
WHERE r.name = 'business_owner'
  AND p.permission_key <> 'platform.manage';

INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r JOIN permissions p
WHERE r.name = 'business_manager'
  AND p.permission_key IN ('staff.manage','services.manage','appointments.manage',
    'customers.manage','sales.manage','expenses.manage','inventory.manage',
    'reports.view','marketplace.manage');

INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r JOIN permissions p
WHERE r.name = 'receptionist'
  AND p.permission_key IN ('appointments.manage','customers.manage','sales.manage');

INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r JOIN permissions p
WHERE r.name = 'professional'
  AND p.permission_key IN ('appointments.manage');

INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r JOIN permissions p
WHERE r.name = 'customer'
  AND p.permission_key IN ('marketplace.manage');

INSERT INTO platform_settings (setting_key, setting_value, description) VALUES
 ('default_currency','RWF','Default currency for Rwanda'),
 ('default_timezone','Africa/Kigali','Default platform timezone'),
 ('default_language','en','Default platform language'),
 ('supported_languages','en,rw,fr','Languages planned or supported by the platform');

-- IMPORTANT IMPLEMENTATION NOTES:
-- 1. Enforce tenant isolation in every backend query using business_id and
--    the authenticated user's active business_memberships. Never trust a
--    business_id supplied by the browser without authorization checks.
-- 2. Prevent overlapping confirmed appointments in a transaction:
--    lock/check the staff's overlapping time range before inserting/updating.
--    MySQL does not provide a simple native exclusion constraint for this.
-- 3. Keep all money as DECIMAL in MySQL and use decimal-safe arithmetic in Node.
-- 4. Update inventory_items.quantity_on_hand and insert stock_movements in
--    the same database transaction.
-- 5. Store password hashes (bcrypt/Argon2), never plaintext passwords.
-- 6. Store timestamps in a consistent server convention and interpret/display
--    business schedules using Africa/Kigali.
