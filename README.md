# ATM System Design with Database Integration

_OBJECTIVE:_ The main aim of this project is to develop a secure ATM system using RFID authentication and a PIN-based interface, with backend banking database integration implemented in C using data structures

__PROJECT OVERVIEW:__
The system consists of two main components:
1. Front-End (Microcontroller (LPC2148) side):\
   Acts like a real ATM interface: 
    - RFID card reader to identify user
    - Keypad for PIN entry and transaction inputs
    - LCD display for menu and status messages
    - Communicates with backend PC via UART
    - Handles the ATM interface (LCD, keypad, RFID reader)

2. Back-End (PC (Linux) side Application in C):\
  Simulates a banking system:  
   - Stores user and transaction data using struct and file I/O
   - Receives commands via UART from MCU
   - Sends results and balance info back to MCU
   - Simulates a banking database using data structures and file handling

___________________________________________________________________________________________________

__BLOCK DIAGRAM:__

![Screenshot 2025-05-23 092351](https://github.com/user-attachments/assets/fbb9cce0-810c-4709-b059-8a5235e3e78e)


 ____________________________________________________________________________________________________________
__HARDWARE REQUIREMENTS:__
1.	LPC2148 Microcontroller: The brain of your ATM system
2.	RFID Reader and Cards: For user authentication
3.	16x2 LCD Display: For user interface
4.	4x4 Matrix Keypad: For PIN and amount input
5.	MAX232: For UART level shifting
6.	Buzzer: For alerts/notifications
7.	USB-to-UART Converter/DB-9 Cable: For communication between MCU and PC

____________________________________________________________________________________________________________________
__SOFTWARE REQUIREMENTS:__ 
1.	Embedded C: For microcontroller programming
2.	Keil uVision IDE: For development environment
3.	Flash Magic: For flashing the program to MCU
4.	GCC Compiler: For PC-side program compilation

________________________________________________________________________________________________________________________
__MICROCONTROLLER SIDE IMPLEMENTATION:__
1.	Hardware Interface Modules:\
Create the Baisc Modules files:
      - lcd.c, lcd.h (LCD control)
      - delay.c, delay.h (timing functions)
      - uart.c, uart.h (UART communication)
      - keypad.c, keypad.h (keypad input)

2.	Testing Individual Modules:\
Before integration, test each module separately:
     - LCD Test: Display characters and strings
     - Keypad Test: Read and display input values
     - UART Test: Send/receive test strings using UART0 and UART1
     - RFID Test: Read card data via UART1 and display on LCD

4.	Main Program Flow (main.c)
1.	Initialization:
      - Initialize all hardware modules (LCD, UART, keypad, RFID)
      - Display project title briefly on LCD
2.	RFID Authentication:
    - Continuously wait for RFID card
    - When card is detected, read the 10-byte frame (starts with 0x02, ends with 0x03)
    - Extract the 8-byte card number (middle bytes)
    - Send to PC in format: #CARD:12345678$
3.	PIN Verification:
       - If PC responds with @OK#VALID_CARDS, display "Enter PIN"
       - Read 4-digit PIN from keypad
       - Send to PC in format: #CARD:12345678#PIN:4321$
       - Wait for validation response
4.	Main Menu:
       - On successful login, display menu options:
         1.	BALANCE
         2.	DEPOSIT
         3.	WITHDRAW
         4.	PIN CHANGE
         5.	MINI STATEMENT
         6.	EXIT
      - Implement 30-second timeout using timer interrupts
5.	Transaction Handling:
      - For each menu option, send appropriate request to PC and handle response

____________________________________________________________________________________________________
__PC-SIDE DATABASE IMPLEMENTATION__

Main Program Flow 
1.	Initialization:
      - Load user data from users.txt into an array/list
      - Load transaction history from transactions.txt into a linked list
      - Initialize UART communication with MCU
2.	RFID Validation:
      - Wait for #CARD:12345678$
      - Search user list for matching RFID
      - Respond with @OK#VALID_CARDS or @ERR#INVALID_CARDS
3.	PIN Validation:
      - Wait for #CARD:12345678#PIN:4321$
      - Verify PIN matches stored PIN for that RFID
      - Respond with @OK#LOGIN_SUCCESS or @ERR#INVALID_PINS
4.	Transaction Handling:
      - Balance Enquiry (#TXN:BALANCE#REQS):\
            - Fetch and return balance: @OK#BALANCE:XXXX.XX$
      - Deposit (#TXN:DEPOSIT#AMOUNT:1000$):\
            - Add amount to balance\
            - Log transaction\
            - Return new balance
      - Withdrawal (#TXN:WITHDRAW#AMOUNT:500$):\
            - Check sufficient balance\
            - Deduct amount if available\
            - Log transaction\
            - Return new balance or error
      - PIN Change (#PINCHANGE#NEWPINS):\
            - Update PIN in user structure\
            - Save to file
      - Mini Statement (#MINISTMT#REQS):\
            - Find last 3 transactions for user\
            - Format as: @OK#MINISTMT:TXN1|TXN2|TXN3$
5.	File Handling:
      - After each transaction that modifies data, update the text files
      - Implement proper file locking to prevent corruption

__________________________________________________________________________________________________________________________________________
  __ATM Communication Protocol: MCU ↔ PC Commands__

| Sr. No. | Transaction Type  | Direction   | Command Format                | Response Format                     | Description |
|---------|-------------------|-------------|-------------------------------|-------------------------------------|-------------|
| 1       | Card Verification | MCU → PC    | `#C:<RFID>$`                  | `@OK:ACTIVE:<Name>$`                | Valid card with account name |
|         |                   |             |                               | `@ERR:BLOCK$`                       | Card is blocked |
|         |                   |             |                               | `@ERR:INVALID$`                     | Card not registered |
| **2**   | PIN Verification  | MCU → PC    | `#V:<RFID>:<PIN>$`            | `@OK:MATCHED$`                      | Correct PIN |
|         |                   |             |                               | `@ERR:WRONG$`                       | Incorrect PIN |
| **3**   | Withdrawal        | MCU → PC    | `#A:WTD:<RFID>:<Amount>$`     | `@OK:DONE$`                         | Withdrawal successful |
|         |                   |             |                               | `@ERR:LOWBAL$`                      | Insufficient balance |
|         |                   |             |                               | `@ERR:NEGAMT$`                      | Negative amount invalid |
|         |                   |             |                               | `@ERR:MAXAMT$`                      | Exceeds ₹30,000 limit |
| **4**   | Deposit           | MCU → PC    | `#A:DEP:<RFID>:<Amount>$`     | `@OK:DONE$`                         | Deposit successful |
|         |                   |             |                               | `@ERR:NEGAMT$`                      | Negative amount invalid |
|         |                   |             |                               | `@ERR:MAXAMT$`                      | Exceeds ₹30,000 limit |
| **5**   | Balance Inquiry   | MCU → PC    | `#A:BAL:<RFID>$`              | `@OK:BAL=<Amount>$`                 | Returns current balance (e.g., `@OK:BAL=25000$`) |
| **6**   | Mini-Statement    | MCU → PC    | `#A:MST:<RFID>:<TxNo>$`       | `@TXN:<Type>:<Date>:<Amount>$`      | Transaction details (type, date, amount) |
|         |                   |             |                               | `@TXN:7$`                           | No more transactions to show |
| **7**   | PIN Change        | MCU → PC    | `#A:PIN:<RFID>:<NewPIN>$`     | `@OK:DONE$`                         | PIN updated successfully |
| **8**   | Card Blocking     | MCU → PC    | `#A:BLK:<RFID>$`              | `@OK:DONE$`                         | Card blocked (after 3 failed PIN attempts) |
| **9**   | System            | MCU → PC    | `#X:LINEOK$`                  | `@X:LINEOK$`                        | Connection test (keep-alive) |
| **10**  | Session End       | MCU → PC    | `#Q:SAVE$`                    | *(None)*                            | Graceful termination (no response expected) |


__________________________________________________________________________________________________________________________________________________________________________________
# ATM Project's Working Flow
__1. System Initialization__
`[MCU BOOT] → [Peripheral Initialization] → [Welcome Screen]`

**Actions:**
- MCU powers on and initializes:
  - UART (for PC communication)
  - LCD (16x2 display)
  - Keypad (4x4 matrix)
  - RFID reader
- Displays: "Welcome To ATM"
- Sends test command: `#X:LINEOK$` to verify PC connection.

---
__2. Card Authentication__
`[RFID Scan] → [Card Validation] → [Block/Proceed]`

**Steps:**
1. User places card → RFID reader detects card and extracts 8-digit ID.
2. MCU sends to PC: `#C:12345678$`
3. PC responds with:
   - Valid: `@OK:ACTIVE:JohnDoe$` (shows name on LCD)
   - Blocked: `@ERR:BLOCK$` → Display "Card Blocked"
   - Invalid: `@ERR:INVALID$` → Display "Card Not Found"

---
__3. PIN Verification__
`[PIN Entry] → [Server Check] → [Access Granted/Block]`

**Process:**
- User enters 4-digit PIN (masked as **** on LCD)
- MCU sends: `#V:12345678:1111$`
- PC checks and replies:
  - Correct: `@OK:MATCHED$` → Proceeds to menu
  - Wrong: `@ERR:WRONG$` → Retry (3 attempts → blocks card)
- Timeout: 30-second inactivity → auto-exit

---
__4. Main Menu Navigation__
`[Menu Display] → [Keypad Input] → [Transaction Execution]`

**Menu Options:**
1. WITHDRAW CASH  
2. DEPOSIT CASH  
3. VIEW BALANCE  
4. MINI STATEMENT  
5. PIN CHANGE  
6. EXIT ATM

**Navigation:**
- A key: Scroll up
- B key: Scroll down
- 1-6 keys: Select option

---
__5. Transaction Processing__

_A. Withdrawal_
`[Amount Entry] → [MCU: #A:WTD:12345678:5000$] → [PC Checks] → [Dispense Cash]`

**Responses:**
- `@OK:DONE$` → Dispenses cash + shows success
- `@ERR:LOWBAL$` → Shows "Low Balance"
- `@ERR:MAXAMT$` → Shows "Exceeds Limit (₹30,000)"

_B. Deposit_
`[Amount Entry] → [MCU: #A:DEP:12345678:10000$] → [PC Updates Balance] → [Confirmation]`

**Responses:** Same as withdrawal (success/error codes)

_C. Balance Inquiry_
`[MCU: #A:BAL:12345678$] → [PC: @OK:BAL=25000$] → [LCD Shows Balance]`

_D. Mini-Statement_
`[MCU: #A:MST:12345678:1$] → [PC: @TXN:1:01/06/2023:5000$] → [LCD Shows Last 3 Transactions]`

_E. PIN Change_
`[Old PIN → New PIN → Confirm] → [MCU: #A:PIN:12345678:2222$] → [PC: @OK:DONE$]`

**Behavior:**
- Failure: Mismatch → decrement tries
- Success: Resets try counter

---
__6. Session Termination__
`[Exit Selected] → [MCU: #Q:SAVE$] → [LCD: "Thank You"] → [Return to Card Scan]`

- Timeout: 30s inactivity → auto-logout with message: "Session Time-Out"

---
__Error Handling & Security__
- **Card Blocking:** After 3 failed PIN attempts → MCU sends `#A:BLK:12345678$`
- **Protocol Security:**
  - All commands use `#/@` framing to prevent corruption
  - Timeouts prevent infinite hangs

_________________________________________________________________________________________________________________________________

# OUTPUTS: 

