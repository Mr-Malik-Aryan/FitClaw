# FitClaw: AI Fitness and Nutrition Coaching Agent - System Design Document
## Architecture Diagram

<img width="2752" height="1568" alt="fitclaw_architecture" src="https://github.com/user-attachments/assets/ddf0f5da-931e-4d3e-baf9-784e9fc93fee" />

### Arrow Color Legend

*   **Blue solid arrow:** Inbound / Fetch data
*   **Green solid arrow:** Outbound / Write / Tool call
*   **Purple solid arrow:** Memory read/write
*   **Amber/orange dashed arrow:** Proactive / Scheduled trigger
*   **Gray thin arrow:** Internal read/write sync

## 1. Introduction

This document outlines the system design for FitClaw, an AI-powered fitness and nutrition coaching agent. FitClaw leverages a layered architecture to provide personalized coaching, track user progress, and integrate with various external services and user channels. The system is designed for scalability, responsiveness, and extensibility, ensuring a robust and adaptive platform for AI-driven health and wellness.

## 2. System Architecture Overview

FitClaw's architecture is composed of distinct, horizontally layered components, facilitating clear separation of concerns and efficient data flow. The system processes user interactions from various channels, leverages memory layers for personalized experiences, employs a sophisticated reasoning agent, and integrates with external services through a Model Context Protocol (MCP) dispatch layer. Proactive scheduling ensures timely and personalized interventions.

## 3. Layered Architecture Details

### 3.1. Layer 1: User Channels

This layer represents the primary interface through which users interact with the FitClaw agent. It encompasses a variety of communication platforms, ensuring broad accessibility and user convenience.

**Components:**

*   **WhatsApp:** A popular messaging application for direct user communication.
*   **Telegram:** Another widely used messaging platform.
*   **FitClaw UI:** Web GUI for managing FitClaw assistant

**Data Flow:**

*   **Inbound:** Messages from all channel nodes flow downwards to Layer 2 (Agent Harness) with the label "message + channel_id".
*   **Outbound:** Replies, alerts, and proactive notifications flow upwards from Layer 2 back to the respective channel nodes. Proactive notifications are indicated by a dashed arrow.

### 3.2. Layer 2: Agent Harness

The Agent Harness acts as a single gateway for all incoming and outgoing communications, centralizing critical functionalities before interactions reach the core reasoning agent.

**Components:**

*   **Agent Harness:** A central component responsible for:
    *   Resolving channel-specific identifiers to unique `user_id`s.
    *   Managing session routing.
    *   Implementing rate limiting to prevent abuse and ensure system stability.
    *   Performing intent pre-parsing to categorize and understand user requests early.
    *   Handling Voice STT outputs.
    *   Routing photo inputs (e.g., for meal or weight tracking).


**Data Flow:**

*   **Inbound:** Receives "message + channel_id" from Layer 1.
*   **Outbound (Memory Fetch):**
    *   Sends "fetch soft memories" to Mem0 (Layer 3) (purple arrow).
    *   Sends "fetch hot session state" to Redis (Layer 3) (amber/orange arrow).

### 3.3. Layer 3: Memory Layer

This layer is responsible for storing and retrieving user-specific data, categorized into soft preferences and hot session state, crucial for personalized coaching.

**Components:**

*   **Mem0 (Soft Preference Memory):**
    *   **Functionality:** Semantic search, utilizing `pgvector` embeddings for efficient retrieval of long-term user preferences.
    *   **Stored Data:** Preferences, habits, dietary restrictions, schedule patterns.

*   **Redis (Hot Session Cache):**
    *   **Functionality:** High-speed, sub-millisecond reads with TTL-based expiry for transient session data.
    *   **Stored Data:** Today's macros, streak information, coach mode, active goals.


**Data Flow:**

*   **Inbound:** Receives requests from Layer 2 (Agent Harness).
*   **Outbound (to Reasoning Agent):**
    *   Mem0 injects "user preferences into system prompt" (purple arrow).
    *   Redis injects "daily snapshot into system prompt" (amber/orange arrow).
    *   Both converge into the Reasoning Agent box (Layer 4).

### 3.4. Layer 4: Reasoning Agent

The core intelligence of FitClaw, this layer processes user input in conjunction with memory data to determine appropriate actions and generate responses.

**Components:**

*   **Reasoning Agent:** The central processing unit, powered by an advanced language model.
    *   **Input:** User profile, Mem0 memories, Redis snapshot, and the user's message.
    *   **Output:** Decides the next action, which could be generating a response, updating memory, or calling an external tool.


**Data Flow:**

*   **Inbound:** Receives converged data from Mem0 and Redis, along with the user message.
*   **Outbound:**
    *   **To Mem0 (Write):** "background: extract new facts from conversation" (dashed purple arrow, indicating an asynchronous process).
    *   **To MCP Dispatch:** "explicit tool call" (solid green arrow).

### 3.5. Layer 5: MCP Dispatch Layer

This layer handles interactions with various external services and internal FitClaw functionalities through the Model Context Protocol (MCP).

**Components:**

*   **FitClaw MCP Server:** Manages core FitClaw actions.
    *   **Functions:** `log_meal`, `log_workout`, `set_goal`, `log_weight`, `resolve_food`.
    *   **Icon:** Database/gear icon.
*   **Swiggy Food MCP:** Integrates with Swiggy's food delivery services.
    *   **Functions:** `search`, `menu`, `cart`, `coupon`, `order`, `track`.
    
*   **Swiggy Instamart MCP:** Integrates with Swiggy's grocery delivery services.
    *   **Functions:** `products`, `go-to items`, `cart`, `checkout`.
    
*   **Swiggy Dineout MCP:** Integrates with Swiggy's restaurant booking services.
    *   **Functions:** `search`, `slots`, `book`, `status`.
    
**Data Flow:**

*   **Inbound:** Receives "explicit tool call" from Layer 4 (Reasoning Agent).
*   **Outbound:**
    *   From FitClaw MCP to Business Logic: "validated write".
    *   From Swiggy MCPs to Business Logic / External: "order confirmation / delivery status".

### 3.5.5. Layer 5.5: Business Logic Layer

Positioned between the MCP Dispatch and Storage layers, this layer enforces business rules and performs computations.

**Components:**

*   **Business Logic Layer:**
    *   **Functionality:** Data validation, computation of derived fields (e.g., calorie burn, streak, macro rebalance), and maintaining an audit trail.

**Data Flow:**

*   **Outbound:** "commit to database" to Layer 6 (Storage).

### 3.6. Layer 6: Storage

This layer centralizes all persistent data storage within a single PostgreSQL instance, with data keyed by `user_id`.

**Components:**

*   **PostgreSQL Instance:** The primary data store.
    *   **fitclaw.* (relational):** Stores user-specific relational data.
        *   **Tables:** `meal_logs`, `workout_logs`, `orders`, `goals`, `streaks`, `weight_logs`, `scheduled_jobs`, `nutrient_tracking`.
    *   **nutrition.* (shared, read-only):** Stores global, read-only nutrition data.
        *   **Tables:** `ingredients`, `dishes`, `recipes`, `recipe_ingredients`, `aliases`.
        *   **Source:** IFCT + USDA sourced data.
    *   **mem0.memories (pgvector):** Stores vector embeddings for user preferences.
        *   **Tables:** `vector embeddings`, `user preferences`, `HNSW index`.
        *   **Icon:** pgvector icon or vector/embedding icon.

**Data Flow:**

*   **Inbound:** Receives "commit to database" from Layer 5.5 (Business Logic).
*   **Internal:** Small bidirectional arrows indicate that the three boxes represent different schemas within the same PostgreSQL instance.
*   **Outbound:** From `fitclaw.scheduled_jobs` to Layer 7: "reads jobs where next_run <= now()".

### 3.7. Layer 7: Proactive Scheduler

This layer is responsible for initiating proactive communications and actions based on scheduled events and user data.

**Components:**

*   **Proactive Scheduler:**
    *   **Functionality:** Utilizes OpenClaw Cron to fire per-user events at the correct local timezone.
    *   **Scheduled Events:** Morning brief, meal reminders, post-workout nutrition, hydration nudges, weekly grocery, evening summary, weekly report.

**Data Flow:**

*   **Inbound:** Reads scheduled jobs from Layer 6 (Storage).
*   **Outbound (Critical Loop):** A prominent, dashed amber/orange arrow loops back UP to the Agent Harness (Layer 2) along the right side of the diagram, labeled "proactive trigger → harness → channels". This signifies the autonomous loop of the FitClaw system.

## 4. Additional System Elements

### 4.1. Food Resolution Engine

This sub-system, attached to the FitClaw MCP, intelligently resolves user-provided food names into detailed nutritional information.

**Flow:**

1.  **User says dish name:** Initial input from the user.
2.  **Fuzzy match (pg_trgm):** Uses `pg_trgm` for fuzzy matching against a database of food items.
3.  **Recipe decomposition:** Breaks down complex dishes into their constituent ingredients.
4.  **Check user history (Mem0):** Consults Mem0 for user-specific preferences or past entries.
5.  **Ask about additions:** Prompts the user for additional ingredients or modifications.
6.  **Compute macros:** Calculates the macronutrient content.
7.  **Log:** Records the meal details.

**Example:** Resolves 'besan cheela' → besan 50g + oil 10g + water 80ml + [onion? cheese?]

### 4.2. Wearable Inputs

External wearable devices integrate with FitClaw via webhooks, providing valuable health and activity data.

**Components:**

*   **Oura Ring:** Provides sleep and activity data.
*   **8Sleep:** Tracks sleep metrics.
*   **Google Fit:** A platform for health and fitness data.
*   **Fitbit:** Wearable fitness trackers.
*   **Garmin:** GPS and wearable technology.
*   **Apple HealthKit:** Apple's health data platform.

**Data Flow:**

*   **Inbound:** "webhook → activity/sleep/HRV data" into the Agent Harness (Layer 2).

### 4.3. Coach Modes

These modes alter the agent's persona and behavior, offering tailored coaching experiences.

**Modes:** BEAST, ZEN, BIO, GAME, FOODIE.

### 4.4. GPT-4o Vision

An integrated AI vision component enhances the agent's ability to process visual information.

**Components:**

*   **GPT-4o Vision:** Utilizes OpenAI's vision capabilities.
    *   **Functionality:**
        *   "meal photo → macro identification": Analyzes meal photos to identify food items and estimate macronutrients.
        *   "scale photo → weight OCR": Performs Optical Character Recognition (OCR) on scale photos to record weight.
        
## 5. Conclusion

This system design document provides a comprehensive overview of the FitClaw AI fitness and nutrition coaching agent. The layered architecture, robust memory management, intelligent reasoning, and integration capabilities ensure a powerful and flexible platform capable of delivering highly personalized and proactive health coaching. The inclusion of advanced features like the Food Resolution Engine, wearable integrations, and AI vision further enhances FitClaw's ability to support users in achieving their fitness and nutrition goals.



