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
__MCU HARDWARE CONNECTIONS__

# LPC2148 Interfacing Connections

## LCD 16x2 ↔ LPC2148

| LCD Pin | LPC2148 Pin | Description        |
|---------|-------------|--------------------|
| D0      | P1.16       | Data Line 0        |
| D1      | P1.17       | Data Line 1        |
| D2      | P1.18       | Data Line 2        |
| D3      | P1.19       | Data Line 3        |
| D4      | P1.20       | Data Line 4        |
| D5      | P1.21       | Data Line 5        |
| D6      | P1.22       | Data Line 6        |
| D7      | P1.23       | Data Line 7        |
| RS      | P0.16       | Register Select    |
| RW      | P0.17       | Read/Write         |
| EN      | P0.26       | Enable             |
| VCC     | 5V          | Power Supply       |
| GND     | GND         | Ground             |

---

## Keypad (4x4) ↔ LPC2148

| Keypad Line | LPC2148 Pin | Description        |
|-------------|-------------|--------------------|
| R0          | P1.24       | Row 0              |
| R1          | P1.25       | Row 1              |
| R2          | P1.26       | Row 2              |
| R3          | P1.27       | Row 3              |
| C0          | P1.28       | Column 0           |
| C1          | P1.29       | Column 1           |
| C2          | P1.30       | Column 2           |
| C3          | P1.31       | Column 3           |

---

## RFID Reader ↔ LPC2148

| RFID Pin        | LPC2148 Pin | Description        |
|------------------|-------------|--------------------|
| TX (output)      | P0.9        | UART1 RX (Receive) |
| VCC              | 5V          | Power Supply       |
| GND              | GND         | Ground             |

---

## PC (via DB9) ↔ LPC2148

| DB9 Pin | LPC2148 Pin | Description        |
|---------|-------------|--------------------|
| RX (Pin 2) | P0.0     | UART0 TX (to PC)   |
| TX (Pin 3) | P0.1     | UART0 RX (from PC) |

---

> 📌 **Note**: Ensure proper voltage level conversion between LPC2148 (3.3V) and peripherals like PC (RS-232 uses ±12V) using MAX232 or level shifter where required.

___________________________________________________________________________________________________________________________________________________________________________________________
# OUTPUTS: 

__FILES__

![Screenshot 2025-06-18 095628](https://github.com/user-attachments/assets/d77a11a3-c8ca-439b-9064-d32b13ac9738)

__Main Projects__

1. _Code Uploading_

![Screenshot from 2025-06-24 10-27-17](https://github.com/user-attachments/assets/3dcd07b5-72d7-484b-9fa8-3e6388f82534)

2. _System Initialization_
   
![WhatsApp Image 2025-06-24 at 11 22 42_86859e1b](https://github.com/user-attachments/assets/97fdbd81-4cf6-44d3-ba47-b900aa024991)

3. _Test without PC_

![Screenshot from 2025-06-24 10-27-01](https://github.com/user-attachments/assets/409caf2d-8fdb-4a3c-bc55-73715a89cde1)

4. _Admin (Bank side Infromation Fill)

![Screenshot from 2025-06-24 10-16-30](https://github.com/user-attachments/assets/4861cc5f-69a3-4fd2-8953-a0174b20b38a)


![Screenshot from 2025-06-24 10-19-16](https://github.com/user-attachments/assets/524c7cd9-b521-4461-ad1f-0e9e14e88ce8)

5. _Card Authentication_

a. Card Invalid 

![IMG-20250624-WA0026](https://github.com/user-attachments/assets/42bc8ca6-ff6c-4328-a206-c07044a65805)

b. Card Block

![IMG-20250624-WA0020](https://github.com/user-attachments/assets/685eba24-11f1-4793-9b2f-ad5ba9b364cb)

c. Valid Card 

![IMG-20250624-WA0019](https://github.com/user-attachments/assets/1758c55c-9df7-4b12-bc9e-86a713f19b02)


6. _Pin Varification_

a. Wrong Pin 

![IMG-20250624-WA0022](https://github.com/user-attachments/assets/d582fb7a-bd8d-4c29-9830-3379b4733545)

b. Correct or Matched Pin 

![IMG-20250624-WA0021](https://github.com/user-attachments/assets/ba7d1919-b090-433e-8884-1ba66ecb8e28)

7. _Menu_

      1. Withdrawal

![IMG-20250624-WA0018](https://github.com/user-attachments/assets/5a00e23a-28c6-47bf-8ce7-f05a4cb15f54)
  
      2. Deposit

![IMG-20250624-WA0016](https://github.com/user-attachments/assets/beefa897-b053-49ba-9abf-4eddb24a6f07)

      3. Balance Inquiry

![IMG-20250624-WA0014](https://github.com/user-attachments/assets/65ca0f53-2b2f-4124-b0b8-bc44292a13f7)

      4. Mini Statement

![IMG-20250624-WA0017](https://github.com/user-attachments/assets/d3e1ac69-60ef-4146-83b2-15f1d807872b)

      5. Pin change
      6. Exit

![IMG-20250624-WA0015](https://github.com/user-attachments/assets/6f36fda8-e39d-42b1-9a60-9587590b9577)


7. _Withdrawal_

a. Success

![IMG-20250624-WA0012](https://github.com/user-attachments/assets/5c54433a-9ed4-413c-8616-98c04b744926)

b. Low Balance

![IMG-20250624-WA0013](https://github.com/user-attachments/assets/68f90179-6161-405d-97bd-59a5e0f49cff)

c. Max Limit 

![IMG-20250624-WA0011](https://github.com/user-attachments/assets/6df863dc-7f99-4abf-bfa6-9112e1230f37)


8. _Deposit Successfully_

![IMG-20250624-WA0029](https://github.com/user-attachments/assets/b128ba1e-d724-4a74-ae96-79d961e10a5d)

9. _Balance Statements_

![IMG-20250624-WA0010](https://github.com/user-attachments/assets/cf959c00-925a-40f9-ad61-9b0f6290bb82)

10. _Mini Statement_

![IMG-20250624-WA0006](https://github.com/user-attachments/assets/341a9fd3-d96a-490b-9f43-9ca3799bf3b8)

![IMG-20250624-WA0007](https://github.com/user-attachments/assets/f4c90993-26f5-471a-b17d-41f802cae9d7)

![IMG-20250624-WA0009](https://github.com/user-attachments/assets/abb06d8b-d96d-48f0-aee2-178ea454da25)

11. _Pin Change_

a. old pin 

![IMG-20250624-WA0004](https://github.com/user-attachments/assets/2e27259e-ce7a-4d91-8810-d384c1ff68b1)

b. New pin 


![IMG-20250624-WA0002](https://github.com/user-attachments/assets/75f8771d-b291-4cae-828c-e13444e40e55)

![IMG-20250624-WA0005](https://github.com/user-attachments/assets/b0625b97-39b0-4469-b683-406e2f1b83fd)

c. Successfully change pin 

![IMG-20250624-WA0008](https://github.com/user-attachments/assets/af6e3be1-da5d-4708-8e44-212cf694cfff)

12. _Exit_


![IMG-20250624-WA0003](https://github.com/user-attachments/assets/1a5fe901-0fcc-4193-b8ad-55c520285259)


_____________________________________________________________________________________________________________________________________
