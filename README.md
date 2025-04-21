# PostgreSQL Event Trigger Extension (pg_eventtrig_ext)

[![License](https://img.shields.io/badge/License-PostgreSQL-blue.svg)](https://www.postgresql.org/about/license/)
[![PostgreSQL Version](https://img.shields.io/badge/PostgreSQL-17%2B-brightgreen.svg)](https://www.postgresql.org/)

A PostgreSQL kernel extension adding three new event trigger types: **logout**, **idle_timeout**, and **shutdown**. This project aims to enhance PostgreSQL's event-driven capabilities for scenarios requiring fine-grained monitoring of user sessions and system lifecycle events. 

## 📌 Key Features

### New Event Triggers

1. **Logout Event Trigger**
   - Triggered when a user explicitly logs out (e.g., via `\c` in `psql` or programmatic disconnection).
   - Use case: Audit user session terminations, log access history, or clean up session-specific resources.
2. **Idle Timeout Event Trigger**
   - Triggered when a database connection is terminated due to inactivity (configured via `idle_session_timeout`).
   - Use case: Monitor and manage idle connections, enforce resource limits, or log timeout events for performance analysis.
3. **Shutdown Event Trigger**
   - Triggered when the PostgreSQL server is shut down (graceful or immediate mode, e.g., `pg_ctl stop`).
   - Use case: Execute pre-shutdown tasks (e.g., data cleanup, final logs), or record system shutdown events for auditing.

## 🛠 Technical Implementation

### Core Modifications

```text
src/include/catalog/pg_database.h        # System table extension (flag columns)
src/backend/commands/event_trigger.c    # Trigger dispatch logic
src/backend/utils/cache/evtcache.c      # Cache management
```

### Architectural Highlights

- **Two-Level Caching**  
  Hybrid in-memory (backend) + persistent (pg_database) flag storage
- **Lock-Free Optimization**  
  ConditionalLockSharedObject for non-blocking flag cleanup
- **Transactional Safety**  
  Atomic updates with CatalogTupleUpdate() and CommandCounterIncrement()

## 🚀 Quick Start

### Build from Source

```bash
# Clone customized PostgreSQL
git clone https://github.com/whupyw/pg_eventtrig_ext
cd pg_eventtrig_ext

# Configure and build
./configure --prefix=/path/to/pgsql
make
make install

# Initialize test cluster
/path/to/initdb -D /path/to/data
/path/to/pg_ctl -D /path/to/data -l logfile start
```

## 📘 Usage Guide

### 1. Create Log Tables (Example)

```sql
-- Logout event log table  
CREATE TABLE user_logouts (  
    id SERIAL PRIMARY KEY,  
    who TEXT NOT NULL,  
    logout_time TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP  
);  

-- Idle timeout log table  
CREATE TABLE idle_timeouts (  
    id SERIAL PRIMARY KEY,  
    who TEXT NOT NULL,  
    timeout_time TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP  
);  

-- Shutdown log table  
CREATE TABLE system_shutdown (  
    id SERIAL PRIMARY KEY,  
    shutdown_time TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP  
);  
```

### 2. Create Trigger Functions

```sql
-- Logout trigger function  
CREATE FUNCTION on_logout_proc() RETURNS EVENT_TRIGGER AS $$  
BEGIN  
    INSERT INTO user_logouts (who) VALUES (SESSION_USER);  
END;  
$$ LANGUAGE plpgsql;  

-- Idle timeout trigger function  
CREATE FUNCTION on_idle_timeout_proc() RETURNS EVENT_TRIGGER AS $$  
BEGIN  
    INSERT INTO idle_timeouts (who) VALUES (SESSION_USER);  
END;  
$$ LANGUAGE plpgsql;  

-- Shutdown trigger function  
CREATE FUNCTION on_shutdown_proc() RETURNS EVENT_TRIGGER AS $$  
BEGIN  
    INSERT INTO system_shutdown (shutdown_time) VALUES (CURRENT_TIMESTAMP);  
END;  
$$ LANGUAGE plpgsql;  
```

### 3. Create Event Triggers

```sql
-- Logout trigger (always enabled)  
CREATE EVENT TRIGGER on_logout_trigger ON LOGOUT EXECUTE PROCEDURE on_logout_proc();  
ALTER EVENT TRIGGER on_logout_trigger ENABLE ALWAYS;  

-- Idle timeout trigger (requires idle_session_timeout configuration)  
CREATE EVENT TRIGGER on_idle_timeout_trigger ON idle_timeout EXECUTE PROCEDURE on_idle_timeout_proc();  
ALTER EVENT TRIGGER on_idle_timeout_trigger ENABLE ALWAYS;  

-- Shutdown trigger  
CREATE EVENT TRIGGER on_shutdown_trigger ON shutdown EXECUTE PROCEDURE on_shutdown_proc();  
ALTER EVENT TRIGGER on_shutdown_trigger ENABLE ALWAYS;  
```

## ✅ Testing Guide

### 1. Regression Testing (Verify No Impact on Native Features)

```bash
make check  # Run PostgreSQL's built-in regression tests  
```

### 2. Functional Testing

#### - Logout Trigger

1. Connect and logout: `/path/to/psql -U your_user -d your_db   ` and `\c` 
2. Verify logs: `SELECT * FROM user_logouts;`

#### - Idle Timeout Trigger

1. Set timeout: `SET idle_session_timeout = 10000;`
2. Leave connection idle for 10s, then check `idle_timeouts` table.

#### - Shutdown Trigger

1. Find the PostgreSQL master process: `ps aux | grep postgres`
2. Stop the server: `kill -INT <postgres_master_pid>`
3. After restart, verify `system_shutdown` table entries.

## 📚 Academic Reference

This project accompanies the thesis *"**Extension and Application of PostgreSQL Event Triggers**"*. 
*Jiawei Cui, Wuhan University, 2025*  
*Advisor: Prof. Yuwei Peng*

## 🤝 Contribution & Feedback

Welcome to submit Issues for bugs/suggestions. For code contributions, create a Pull Request with clear descriptions.

## 📧 Contact

For technical questions or collaborations, email `3086764005@qq.com`.

## 📜 License & Documentation

PostgreSQL License (Same as PostgreSQL base code). Copyright and license information can be found in the file COPYRIGHT. 

For detailed information on PostgreSQL's event trigger mechanism, refer to the official PostgreSQL documentation. General documentation about this version of PostgreSQL can be found at https://www.postgresql.org/docs/17/. In particular, information about building PostgreSQL from the source code can be found at https://www.postgresql.org/docs/17/installation.html. 

