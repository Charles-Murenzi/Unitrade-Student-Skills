# Unitrade
is a digital platform enabling university students to monetize their expertise (e.g., tutoring, design, coding) by connecting with clients for freelance opportunities as a side hustle while balancing academic commitments.
-- Create Database
CREATE DATABASE unitrade_student_skills_db;
USE unitrade_student_skills_db;

-- Table 1: Users
CREATE TABLE users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(10) NOT NULL,
    is_student BOOLEAN NOT NULL DEFAULT FALSE,
    community_credits DECIMAL(10, 2) DEFAULT 0.00 CHECK (community_credits >= 0),
    CONSTRAINT chk_phone_length CHECK (LENGTH(phone) = 10)
);

-- Table 2: Listings (RWF Focused)
CREATE TABLE listings (
    listing_id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    title VARCHAR(100) NOT NULL,
    description TEXT NOT NULL,
    skill_category ENUM('Coding', 'Design', 'Tutoring', 'Writing', 'Translation', 'Other') NOT NULL,
    payment_type ENUM('Cash (RWF)', 'Credits') NOT NULL,
    amount_RWF DECIMAL(10, 2) CHECK (amount_RWF >= 1000), -- Minimum 1000 RWF
    status ENUM('Open', 'Closed') DEFAULT 'Open',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    CONSTRAINT chk_cash_listing CHECK (
        (payment_type = 'Cash (RWF)' AND amount_RWF >= 1000) OR 
        (payment_type = 'Credits')
    )
);

-- Table 3: Transactions (Simplified)
CREATE TABLE transactions (
    transaction_id INT AUTO_INCREMENT PRIMARY KEY,
    listing_id INT NOT NULL,
    buyer_id INT NOT NULL,
    seller_id INT NOT NULL,
    payment_method ENUM('Cash (RWF)', 'Credits') NOT NULL,
    amount_RWF DECIMAL(10, 2) CHECK (amount_RWF IS NULL OR amount_RWF >= 1000), -- Nullable for credits
    status ENUM('Pending', 'Completed', 'Disputed') DEFAULT 'Pending',
    transaction_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    dispute_reason TEXT,
    FOREIGN KEY (listing_id) REFERENCES listings(listing_id) ON DELETE CASCADE,
    FOREIGN KEY (buyer_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (seller_id) REFERENCES users(user_id) ON DELETE CASCADE,
    CONSTRAINT chk_buyer_seller CHECK (buyer_id != seller_id)
);

-- Trigger: Block students from cash listings
DELIMITER $$
CREATE TRIGGER trg_block_student_cash_listings
BEFORE INSERT ON listings
FOR EACH ROW
BEGIN
    IF NEW.payment_type = 'Cash (RWF)' AND 
       (SELECT is_student FROM users WHERE user_id = NEW.user_id) = TRUE THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Students cannot create cash (RWF) listings!';
    END IF;
END$$
DELIMITER ;

-- Event: Auto-close old listings
DELIMITER $$
CREATE EVENT expire_old_listings
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    UPDATE listings
    SET status = 'Closed'
    WHERE status = 'Open' AND created_at < (NOW() - INTERVAL 12 HOUR);
END$$
DELIMITER ;
-- Indexes for Optimization
CREATE INDEX idx_user_email ON users(email);
CREATE INDEX idx_listing_status ON listings(status);
CREATE INDEX idx_transaction_status ON transactions(status);
