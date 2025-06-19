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


__BLOCK DIAGRAM:__

![Screenshot 2025-05-23 092351](https://github.com/user-attachments/assets/fbb9cce0-810c-4709-b059-8a5235e3e78e)

 
__HARDWARE REQUIREMENTS:__
1.	LPC2148 Microcontroller: The brain of your ATM system
2.	RFID Reader and Cards: For user authentication
3.	16x2 LCD Display: For user interface
4.	4x4 Matrix Keypad: For PIN and amount input
5.	MAX232: For UART level shifting
6.	Buzzer: For alerts/notifications
7.	USB-to-UART Converter/DB-9 Cable: For communication between MCU and PC

__SOFTWARE REQUIREMENTS:__ 
1.	Embedded C: For microcontroller programming
2.	Keil uVision IDE: For development environment
3.	Flash Magic: For flashing the program to MCU
4.	GCC Compiler: For PC-side program compilation

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
5.	File Handling:\
      - After each transaction that modifies data, update the text files\
      - Implement proper file locking to prevent corruption

  __ATM Communication Protocol: MCU ↔ PC Commands__

| Sr. No. | Transaction Type  | Direction   | Command Format               | Response Format                     | Description |
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
| **6**   | Mini-Statement    | MCU → PC    | `#A:MST:<RFID>:<TxNo>$`      | `@TXN:<Type>:<Date>:<Amount>$`      | Transaction details (type, date, amount) |
|         |                   |             |                               | `@TXN:7$`                           | No more transactions to show |
| **7**   | PIN Change        | MCU → PC    | `#A:PIN:<RFID>:<NewPIN>$`     | `@OK:DONE$`                         | PIN updated successfully |
| **8**   | Card Blocking     | MCU → PC    | `#A:BLK:<RFID>$`              | `@OK:DONE$`                         | Card blocked (after 3 failed PIN attempts) |
| **9**   | System            | MCU → PC    | `#X:LINEOK$`                  | `@X:LINEOK$`                        | Connection test (keep-alive) |
| **10**  | Session End       | MCU → PC    | `#Q:SAVE$`                    | *(None)*                            | Graceful termination (no response expected) |

