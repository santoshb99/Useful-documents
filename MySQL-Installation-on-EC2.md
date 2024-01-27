# Commands used to host MySql Server on AWS EC2 Instance - Ubuntu 22.04

## Step 1: Update the system
```bash
sudo apt update
```

## Step 2: Install MySql
```bash
sudo apt install mysql-server
```

Or if you want to install a specific version of mysql,
```bash
sudo apt install mysql-server-8.0
```

Enable MySQL to Start on Boot:
```bash
sudo systemctl enable mysql
```

## Step 3: Check the Status of MySql (Active or Inactive)
```bash
sudo systemctl status mysql
```

## Step 4: Login to MySql as a root
```bash
sudo mysql
```

## Step 5: Update the password for the MySql Server
```bash
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'place-your-password-here';
```
```bash
FLUSH PRIVILEGES;
```

## Step 6: Test the MySql server if it is working by running sample sql queries
```bash
CREATE DATABASE mysql_test;
USE mysql_test;
```
```bash
CREATE TABLE table1 (id INT, name VARCHAR(45));
INSERT INTO table1 VALUES(1, 'Virat'), (2, 'Sachin'), (3, 'Dhoni'), (4, 'ABD');
```
```bash
SELECT * FROM table1;
```
---
### Connect to MySQL Server again:
```bash
mysql -u root -p
```
Enter the MySQL root password when prompted.

### (Optional) Update MySQL Password Plugin:
If you encounter authentication issues, you may need to update the MySQL password plugin:
```bash
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'your_password';
FLUSH PRIVILEGES;
```

### To import sql file 
Use the cd command to navigate to the directory where your SQL file is located.
```bash
cd path/to/your/sql/file
```

Run the MySQL Import Command:
```bash
mysql -u root -p < your_sql_file.sql
```
---

