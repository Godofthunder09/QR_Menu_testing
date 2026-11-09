# QR Menu App: test report

A table-QR ordering app (React, Vite, Tailwind, Supabase). Customers scan a code at the table, enter a PIN and order from their phone. The owner gets the orders on a dashboard, prints bills and checks reports.

| | |
|---|---|
| Test cases | 83 in 11 modules |
| Type of testing | Manual: functional, integration and a bit of basic security |
| Roles | Customer (table PIN) and admin/owner |
| Author | Yash Patil |
| Date | 7 Nov 2026 |

## What this is

I wrote these test cases for my QR menu project and used them to review the first version of the code (v1). This report has the test cases, the bugs I found in v1 and where each one stands now.

The test cases describe how the app should behave, so you can run them against any version. I found the bugs by reading the v1 code and checked the fixes by comparing v1 with the current code. Each fix has a test case that checks it on a running build.

## How I tested

Mostly by thinking about what could go wrong at a real table: wrong PIN, a table cleared while a guest is still ordering, a discount typed as 150%, a printer that is switched off. For limits like discounts, image size and the 10-table cap I tested the edges (0, 1, 99, 100). Several cases need two screens at once, a phone as the customer and a laptop as the admin.

Not covered: load testing, automated tests and a full browser/device matrix.

## Numbers

| Module | Cases | High priority |
|---|---:|---:|
| Customer Login (Table PIN) | 8 | 6 |
| Menu Browsing & Search | 6 | 3 |
| Ordering & Cart | 10 | 7 |
| Order Confirmation & My Orders | 6 | 3 |
| Admin Login | 6 | 4 |
| Table Management | 7 | 5 |
| Menu Management | 7 | 3 |
| Billing (Dashboard) | 7 | 6 |
| Kitchen Order Tickets (KOT) | 9 | 5 |
| Reports & Settlement | 10 | 7 |
| Settings & Admin Code | 7 | 3 |
| Total | 83 | 52 |

Results from my test run:

| Run | Passed | Failed | Blocked | Not run |
|---:|---:|---:|---:|---:|
| - | - | - | - | 83 |

## Bugs I found in v1

Nine in total: 2 critical, 5 high, 1 medium, 1 low. Critical means security or data exposure, high means wrong money or lost data.

| ID | Severity | What was wrong | Why it mattered | What changed | Status | Re-test |
|---|---|---|---|---|---|---|
| DEF-01 | Critical | Admin pages opened without logging in. If you typed /admin/dashboard in the address bar you got straight in. | Anyone with the link could read sales and change tables, menu and settings. | All admin pages now sit behind the login check. | Fixed | AL_04 |
| DEF-02 | Critical | The Admin Code (the one that unlocks bill editing and deleting) was written inside the app's JavaScript and checked in the browser. | Anyone could read it from the page source and then edit or delete paid bills. | The code is now checked in the database against a salted hash. The browser never sees it. | Fixed | ST_04 |
| DEF-03 | High | Paid bills could be edited or deleted and nothing recorded who did it. | No trail for changes to money data. | Each edit or delete first writes an audit entry (who, when, what the bill looked like before). If that write fails, the change is stopped. | Fixed | ST_05, ST_06 |
| DEF-04 | High | After a table was cleared, a customer who hadn't been logged out yet could still place an order, and it landed on the next guest's bill. | Wrong guest charged. | Ordering now checks the customer's own session against the table. If it doesn't match they go back to the PIN screen. | Fixed | OC_05 |
| DEF-05 | High | An order was saved in two steps. If the second step failed, an empty order stayed on the dashboard. | Ghost orders for the kitchen and cashier. | The empty order is deleted and the customer is told it wasn't placed. | Fixed | OR_09 |
| DEF-06 | High | Deleting a table also deleted all its orders, paid ones included. | Revenue disappeared from Reports. | Deleting now just hides the table. Paid data stays, customers are logged out, and it refuses if there are unpaid orders. | Fixed | TM_06, TM_07 |
| DEF-07 | Medium | Generating a new PIN didn't log out phones that had already used the old one. | A guest could keep ordering after the PIN changed. | A new PIN now also resets the table session, so those phones go back to the PIN screen within a few seconds. I added a confirm popup too. | Fixed | TM_04 |
| DEF-08 | High | The discount box accepted over 100%, negative numbers and flat amounts bigger than the bill. | Negative or inflated totals. | Percent must be 1 to 99 and flat must be below the payable total, on the dashboard and in Today's Report. Bad input shows a message and blocks Print & Save. | Fixed | BL_04, RP_10 |
| DEF-09 | Low | Reports stopped each day at 23:59:59, so a bill saved in the last second could be missed. | Tiny mismatch at midnight. | Fixed on the Reports page only. Today's Report and Admin Code still use the old cut-off, so this one is only half done. | Half fixed | RP_04 |

"Fixed" means the fix is in the current code. These came from code review, not from running the app, so the re-test column is where I confirm each one on a live build.

## Test cases

Click a module to open it. Status: not run, pass, fail or blocked.

<details>
<summary><b>Customer Login (Table PIN)</b> (8)</summary>

| ID | Scenario | Pre-condition | Steps | Expected result | Priority | Ref | Status |
|---|---|---|---|---|---|---|---|
| CL_01 | Login with valid PIN | Table 1 exists, PIN = 4821 (example), fresh browser session (localStorage cleared) | 1. Scan QR / open menu URL<br>2. Wait for splash screens (Welcome ➜ Logo, ~4 sec)<br>3. On 'Enter Table PIN' screen tap 4 8 2 1<br>4. Tap '✓ OK' | Menu page opens with title 'Our Menu', table name 'Table 1', category pills and food items. Keys pin_session_<id> and ver_<id> are saved in localStorage. | High | - | Not run |
| CL_02 | Login with wrong PIN | Table 1 exists, PIN = 4821 (example), fresh browser session (localStorage cleared) | 1. Open menu URL<br>2. Enter 0000 on keypad<br>3. Tap '✓ OK' | Error 'Wrong PIN. Ask your waiter.' is shown, PIN boxes are cleared, user stays on PIN screen, menu is NOT loaded. | High | - | Not run |
| CL_03 | Login with incomplete PIN | Table 1 exists, PIN = 4821 (example), fresh browser session (localStorage cleared) | 1. Open menu URL<br>2. Enter only 2 digits (4, 8)<br>3. Tap '✓ OK' | Error 'Enter 4 digit PIN' is shown. No request is made to verify the PIN. | Medium | - | Not run |
| CL_04 | QR without table id | None | 1. Open https://<app-url>/menu (no ?table=)<br>2. Wait for splash screens | Screen with ❌ and message 'Invalid QR Code.' is shown. No PIN screen, no menu. | High | - | Not run |
| CL_05 | QR with non-existent / deleted table | A table was created and then deleted by admin (or use random UUID) | 1. Open /menu?table=<deleted-or-random-id><br>2. Wait for splash<br>3. Enter any 4-digit PIN<br>4. Tap '✓ OK' | User must not be able to enter the menu. Message 'Table not found.' (or Invalid QR) is shown. | High | - | Not run |
| CL_06 | Session persists after refresh | Customer already logged in successfully (CL_01) | 1. Press browser refresh (F5)<br>2. Wait for splash screens | Menu loads directly without PIN screen; previously ordered items (My Orders badge) are restored. | High | - | Not run |
| CL_07 | Session invalidated when admin clears table | Customer logged in on phone; admin logged in on Dashboard with Table 1 open | 1. Customer: add 1 item to cart (do not order)<br>2. Admin: bill & clear Table 1 (session_version increments, PIN regenerates)<br>3. Customer: wait up to 5 seconds without touching the screen | Within ~5 sec customer is returned to PIN screen, cart is emptied, localStorage session keys are removed. Old PIN no longer works. | High | - | Not run |
| CL_08 | PIN keypad input limits | Table 1 exists, PIN = 4821 (example), fresh browser session (localStorage cleared) | 1. Open PIN screen<br>2. Tap 1 2 3 4 5 6<br>3. Tap ⌫ twice<br>4. Tap 9 | After 6 taps only 4 boxes are filled (1234). After 2x ⌫ → '12'. After tapping 9 → '129'. Boxes show ● for filled, ○ for empty. | Low | - | Not run |

</details>

<details>
<summary><b>Menu Browsing & Search</b> (6)</summary>

| ID | Scenario | Pre-condition | Steps | Expected result | Priority | Ref | Status |
|---|---|---|---|---|---|---|---|
| MB_01 | Menu list display | Logged in as customer; at least 2 categories with items, one item with image and one without | 1. Login with valid PIN<br>2. Observe header and item list | Each item card shows image (or 🍴 placeholder), name, category › subcategory, description (max 2 lines), price with ₹ and an '+ Add' / '📝 Customize' button. 'All' pill is selected by default. | High | - | Not run |
| MB_02 | Filter by category | Menu has at least 2 categories with different items | 1. Tap a category pill (e.g. 'Starters')<br>2. Tap 'All' | Only items of the selected category are shown; selected pill turns orange. 'All' shows every available item again. | High | - | Not run |
| MB_03 | Search with suggestions | Menu contains item 'Chicken Tikka' | 1. Type 'ch' in search box<br>2. Observe dropdown<br>3. Tap the 'Chicken Tikka' suggestion | Dropdown shows up to 5 items containing 'ch' (case-insensitive). Selecting one fills the box and list filters to matching items. Category pills are hidden while searching. | Medium | - | Not run |
| MB_04 | Search with no match | Logged in as customer | 1. Type 'zzzzxyz' in search box | No suggestions; list shows 🍴 'No items found'. | Medium | - | Not run |
| MB_05 | Unavailable item hidden | Item 'Paneer Tikka' visible on menu; admin account available | 1. Admin: Menu Manager ➜ click '⛔ Hide' on Paneer Tikka<br>2. Customer: refresh menu page<br>3. Search for 'Paneer' | Paneer Tikka is not listed in 'All', its category, or search results. After admin taps '✅ Show' and customer refreshes it reappears. | High | - | Not run |
| MB_06 | Image zoom | Item with image_url exists | 1. Tap food image<br>2. Tap × (or outside) to close | Image opens in a modal overlay; × / backdrop click closes it. | Low | - | Not run |

</details>

<details>
<summary><b>Ordering & Cart</b> (10)</summary>

| ID | Scenario | Pre-condition | Steps | Expected result | Priority | Ref | Status |
|---|---|---|---|---|---|---|---|
| OR_01 | Add item to cart | Customer logged in; empty cart | 1. Tap '+ Add' on 'Chicken Tikka' (₹350) | Card switches to − 1 + stepper. Header shows '🛒 Cart' with badge 1. Bottom bar shows '🛒 1 item in cart / View Cart →'. | High | - | Not run |
| OR_02 | Increase / decrease quantity | Chicken Tikka in cart (qty 1) | 1. Tap + three times<br>2. Tap − once<br>3. Open cart | Quantity = 3. Cart line shows '₹350 × 3 = ₹1050' and subtotal ₹1050; badge shows 3. | High | - | Not run |
| OR_03 | Decrement to zero removes item | Item in cart with qty 1 | 1. Tap − on the item | Item removed from cart; card returns to '+ Add' state; cart badge and bottom bar disappear when cart is empty. | High | - | Not run |
| OR_04 | Add item with customization note | Empty cart | 1. Tap '📝 Customize' on an item<br>2. Type 'Less spicy, no onion'<br>3. Tap 'Add to Cart' | Item added with qty 1 and note displayed as 📝 "Less spicy, no onion" on card and in cart. | High | - | Not run |
| OR_05 | Edit note from cart drawer | Item with note in cart | 1. Tap '🛒 Cart'<br>2. Tap '✏️ Edit customization'<br>3. Change note to 'Extra sauce'<br>4. Tap 'Update Note' | Note modal opens pre-filled with old note; after update the new note is shown in cart. Qty is unchanged. | Medium | - | Not run |
| OR_06 | Cart subtotal with multiple items | Menu has items priced ₹350 and ₹40 | 1. Add Chicken Tikka ×2 (₹350)<br>2. Add Naan ×4 (₹40)<br>3. Open cart | Subtotal shows ₹860 and '(6 items)'. Footer text 'Final bill at counter includes service charges' is visible. | High | - | Not run |
| OR_07 | Remove item using ✕ | Cart has Chicken Tikka ×3 and Naan ×2 | 1. Open cart<br>2. Tap ✕ next to Chicken Tikka | Chicken Tikka removed entirely; Naan ×2 stays; subtotal recalculated to ₹80. | Medium | - | Not run |
| OR_08 | Place order successfully | Cart has at least one item; admin Dashboard open in another browser | 1. Open cart<br>2. Tap '🍽️ Place Order' | Button shows 'Placing order...' then app navigates to /order-confirmation. Cart is emptied. Admin Dashboard shows new pending order for Table 1 (with note) within ~4 sec and plays alert sound if sound enabled. | High | - | Not run |
| OR_09 | Double tap on Place Order | Cart has items; throttle network to 'Slow 3G' in DevTools | 1. Open cart<br>2. Tap 'Place Order' 3–4 times quickly | Only ONE order is created in orders table; button is disabled while placing. | High | DEF-05 | Not run |
| OR_10 | Special characters / very long note | Customer logged in | 1. Tap '📝 Customize' on an item<br>2. Enter note: <script>alert(1)</script> 🌶️ " ' & plus 500 characters<br>3. Add to cart and place order<br>4. Check admin Dashboard order card | No script executes anywhere (menu, My Orders, Dashboard, receipt print). Text is displayed as plain text; emoji preserved; long note wraps without breaking layout. | Medium | - | Not run |

</details>

<details>
<summary><b>Order Confirmation & My Orders</b> (6)</summary>

| ID | Scenario | Pre-condition | Steps | Expected result | Priority | Ref | Status |
|---|---|---|---|---|---|---|---|
| OC_01 | Confirmation page auto-redirect | Order just placed (OR_08) | 1. Observe confirmation page<br>2. Do nothing for 5 seconds | Page shows 🎉 'Order Placed!', table name, and countdown 5s ➜ 0, then redirects to /menu?table=<id> with session still active (no PIN prompt). | High | - | Not run |
| OC_02 | 'Order More Items' button | On confirmation page | 1. Tap '+ Order More Items' before countdown ends | Navigates immediately to menu page of the same table; countdown timer does not trigger a second navigation. | Medium | - | Not run |
| OC_03 | My Orders drawer accumulates quantities | Customer ordered Chicken Tikka ×2, then in a second order Chicken Tikka ×1 + Naan ×2 | 1. Tap '📋 Orders' button in header<br>2. Check list and total | Drawer shows 'Ordered this session: 5 items / 2 dishes'; Chicken Tikka × 3, Naan × 2; notes shown; no prices (billing at counter). | High | - | Not run |
| OC_04 | 'Ordered ×N' badge on item card | Chicken Tikka ordered ×3 in this session | 1. Go to menu list<br>2. Find Chicken Tikka card | Green badge 'Ordered ×3' appears on the card. Items not ordered have no badge. | Low | - | Not run |
| OC_05 | Order after table was cleared by admin | Customer has items in cart; admin clears the table (new session_version + new PIN) before customer taps Place Order | 1. Customer: add items to cart<br>2. Admin: bill & clear table<br>3. Customer: immediately (within 5s polling gap) tap 'Place Order' | Order is rejected and customer is asked to enter the new PIN. No order is created under the new session. | High | DEF-04 | Not run |
| OC_06 | Confirmation page without parameters | None | 1. Open https://<app-url>/order-confirmation directly<br>2. Wait 5 seconds | Page should handle missing table gracefully (e.g. show message / go to invalid-QR screen). | Low | - | Not run |

</details>

<details>
<summary><b>Admin Login</b> (6)</summary>

| ID | Scenario | Pre-condition | Steps | Expected result | Priority | Ref | Status |
|---|---|---|---|---|---|---|---|
| AL_01 | Login with valid credentials | Admin user exists in Supabase Auth | 1. Open https://<app-url>/admin<br>2. Enter valid email and password<br>3. Click 'Login →' | Button shows 'Logging in...' then user lands on /admin/dashboard. | High | - | Not run |
| AL_02 | Login with wrong password | Admin user exists | 1. Open /admin<br>2. Enter valid email and wrong password<br>3. Click 'Login →' | Red message 'Invalid email or password!' appears; user stays on login page; button re-enabled. | High | - | Not run |
| AL_03 | Login with empty fields | On login page | 1. Leave email and password empty<br>2. Click 'Login →' | Login is not successful; error 'Invalid email or password!' (or field validation) is displayed. No navigation. | Medium | - | Not run |
| AL_04 | Direct URL access without login | Not logged in (incognito window) | 1. Open https://<app-url>/admin/dashboard directly<br>2. Repeat for /admin/menu, /admin/tables, /admin/reports, /admin/today-report, /admin/settings, /admin/admin-code | Every URL redirects to the login page ('/'). No admin data is visible. | High | DEF-01 | Not run |
| AL_05 | Logout | Logged in as admin | 1. Click Logout on Dashboard<br>2. Press browser Back button | Redirected to login page. Back button must not reveal dashboard data. | High | - | Not run |
| AL_06 | Submit with Enter key | On login page | 1. Enter valid email and password<br>2. Press Enter in the password box | Login is submitted like clicking the button. | Low | KL-04 | Not run |

</details>

<details>
<summary><b>Table Management</b> (7)</summary>

| ID | Scenario | Pre-condition | Steps | Expected result | Priority | Ref | Status |
|---|---|---|---|---|---|---|---|
| TM_01 | Add a table | Admin logged in; fewer than 10 tables | 1. Go to Tables<br>2. Enter 'Table 5'<br>3. Click 'Add Table' (or press Enter) | Message 'Table added! ✅'. New card appears with 4-digit PIN (masked ••••), QR code and URL ending in /menu?table=<id>. Counter 'x/10 used' increases. | High | - | Not run |
| TM_02 | Add table with empty name | Admin on Tables page | 1. Leave name blank<br>2. Click 'Add Table' | Alert 'Enter table name!' and no table created. | Medium | - | Not run |
| TM_03 | Beta limit of 10 tables | 10 tables already exist | 1. Type 'Table 11'<br>2. Observe Add button<br>3. Try pressing Enter in the input | Add button is disabled, red text 'Beta limit: max 10 tables.' shown; Enter key shows alert 'Beta limit: max 10 tables!' and nothing is inserted. | Medium | KL-06 | Not run |
| TM_04 | Regenerate PIN and sign out old devices | Table 1 exists; a customer phone is logged in to Table 1 with the current PIN | 1. On the Tables page note the PIN shown on the Table 1 card<br>2. Click '🔄 New PIN' and confirm the popup<br>3. Watch the customer phone for up to 5 seconds<br>4. On a new device try the OLD PIN, then the NEW PIN | A different 4-digit PIN is shown on the card. The logged-in customer phone returns to the PIN screen within ~5 s and unpaid orders stay on the bill. The old PIN is rejected; the new PIN opens the menu. | High | DEF-07 | Not run |
| TM_05 | Download QR code | Table exists | 1. Click '⬇️ Download QR'<br>2. Open the PNG<br>3. Scan with mobile camera | File 'QR-<table name>.png' downloads. Scanning opens /menu?table=<id> of the correct table and shows splash ➜ PIN screen. | High | - | Not run |
| TM_06 | Delete a table (soft delete) | Table 4 has at least one PAID bill today and no unpaid orders; a customer phone is logged in to Table 4 | 1. Click 🗑️ on Table 4 and press Cancel<br>2. Click 🗑️ again and confirm<br>3. Open Reports ➜ Today and Table-wise<br>4. Check the customer phone | Cancel keeps the table. Confirm removes Table 4 from the list with the message 'Payment data kept'. Reports still show Table 4's paid bills and the same revenue. The customer phone is sent back to the PIN screen and the old PIN no longer works. | High | DEF-06 | Not run |
| TM_07 | Delete blocked while unpaid orders exist | Table 2 has an unpaid order on the Dashboard | 1. On the Tables page click 🗑️ on Table 2 | Alert says the table still has unpaid orders and to print/settle its bill first. The table and its order are unchanged. | High | DEF-06 | Not run |

</details>

<details>
<summary><b>Menu Management</b> (7)</summary>

| ID | Scenario | Pre-condition | Steps | Expected result | Priority | Ref | Status |
|---|---|---|---|---|---|---|---|
| MM_01 | Add category and food item | Admin logged in | 1. Menu ➜ add category 'Starters'<br>2. Add food item with name, price, category, description<br>3. Save<br>4. Open customer menu | Category and item saved with success message. Customer menu shows the item under 'Starters' with ₹120. | High | - | Not run |
| MM_02 | Add item with invalid data | Admin on Menu Manager | 1. Leave name empty / price empty / price = -50 / price = 0<br>2. Click add item | Item is not saved; validation message is shown for every invalid case. Negative or zero price is not accepted. | High | - | Not run |
| MM_03 | Upload image over 5 MB | Add/Edit item form open; a 6 MB JPG available | 1. Click image upload<br>2. Select the 6 MB image | Error 'Image too large — max 5MB.' is shown and image is not uploaded. A 2 MB JPG uploads and previews successfully. | Medium | - | Not run |
| MM_04 | Toggle availability | Item available | 1. Click '⛔ Hide' on the item<br>2. Check customer menu<br>3. Click '✅ Show' | Message "<item>" hidden ✅ / shown ✅. Card turns grey while hidden; customer sees item only when available. | High | - | Not run |
| MM_05 | Bulk upload via Excel | Excel file with columns for category, subcategory, name, price – 3 valid rows, 1 row without price | 1. Click Excel upload<br>2. Select file<br>3. Review preview and errors<br>4. Click upload | Preview lists valid rows and shows an error for the invalid row. After upload 3 items added, categories/subcategories auto-created without duplicates (case-insensitive), invalid row skipped. | Medium | - | Not run |
| MM_06 | Delete category / bulk delete items | Category with items exists; 3 items selected using checkboxes | 1. Delete the category that has items<br>2. Select 3 items ➜ Bulk Delete ➜ confirm | Category in use is blocked or warned (items not orphaned). Bulk delete asks confirmation listing names, removes items from menu but keeps order history. | Medium | - | Not run |
| MM_07 | Assign a department to an item | Admin on Menu Manager; at least 2 items | 1. Add an item and choose Department = Tandoor ➜ save<br>2. Edit another item and set Department = Billing<br>3. Filter the list by department<br>4. Select 2 items ➜ bulk set department to Beverage | Department is saved and shown as a coloured badge on each item. The filter lists only items of the chosen department. Bulk change updates both selected items. | Medium | - | Not run |

</details>

<details>
<summary><b>Billing (Dashboard)</b> (7)</summary>

| ID | Scenario | Pre-condition | Steps | Expected result | Priority | Ref | Status |
|---|---|---|---|---|---|---|---|
| BL_01 | New order notification | Admin on Dashboard, sound enabled; customer logged in | 1. Customer places order for Table 1<br>2. Watch Dashboard | Within ~4 sec Table 1 is highlighted as new, the order with items and notes is listed, and a two-tone bell plays. | High | - | Not run |
| BL_02 | Bill total with service charge and discount | Table 1 has items totalling ₹1000 | 1. Open billing for Table 1<br>2. Select Service Charge 10%<br>3. Discount type % ➜ value 10<br>4. Enter discount reason<br>5. Observe totals | Service = ₹100, amount after service ₹1100, discount = ₹110, Final = ₹990 (discount is applied on amount after service charge). | High | - | Not run |
| BL_03 | Discount without reason | Billing screen open | 1. Enter discount value 50<br>2. Leave reason blank<br>3. Click Print & Save | Bill is NOT saved/printed; reason field shows error highlight. | High | - | Not run |
| BL_04 | Discount range validation | Billing screen open; bill total ₹500 (no service charge) | 1. Choose % and enter 0, 100, 150, -10, abc, then 25<br>2. Choose Flat and enter 500, 800, then 499<br>3. Observe the message, the total and the Print & Save button each time | Percent is accepted only from 1 to 99 (25% ➜ total ₹375). Flat must be at least ₹1 and less than the payable total (499 accepted; 500 and 800 rejected). Invalid input shows a red message, Print & Save is disabled, and the final amount is never ₹0 or negative. | High | DEF-08 | Not run |
| BL_05 | Add open (manual) item with invalid price | Billing screen open | 1. Click Add Open Item<br>2. Leave name blank ➜ Add<br>3. Enter name, price 0 ➜ Add<br>4. Enter name 'Soda', price 40, qty 2 ➜ Add | Alerts 'Enter item name' and 'Enter valid price' for invalid input. Valid item is added in the Food/Beverage/Liquor department and bill increases by ₹80. | Medium | - | Not run |
| BL_06 | Print & Save – table reset | Table 1 has unpaid orders; customer logged in on phone | 1. Edit a quantity (reduce Chicken Tikka 4 ➜ 2)<br>2. Choose payment method<br>3. Click Print & Save<br>4. Check Reports<br>5. Check customer phone | Receipt prints with restaurant details. Orders become paid with correct subtotal/final; order_items quantity = 2 (matches bill). Table clears, PIN regenerates, customer is sent to PIN screen in ≤5s. Entry appears in Reports. | High | - | Not run |
| BL_07 | Remove last item – End Session | Table 1 has one unpaid order containing a single item; customer phone logged in | 1. Open Table 1 on the Dashboard<br>2. Press − until the only item reaches 0<br>3. Press Cancel in the popup, then repeat and press OK | A popup says the item is the only one left, the session will end and no bill will be printed or saved. Cancel keeps the item. OK clears the table, the customer returns to the PIN screen and no ₹0 bill appears in Reports. | High | - | Not run |

</details>

<details>
<summary><b>Kitchen Order Tickets (KOT)</b> (9)</summary>

| ID | Scenario | Pre-condition | Steps | Expected result | Priority | Ref | Status |
|---|---|---|---|---|---|---|---|
| KT_01 | Print to department printers | QZ Tray is running on the billing PC; in Dashboard ➜ 🖨️ Printers a printer name is set for Kitchen and Beverage; items have departments assigned | 1. Customer orders 1 Kitchen item and 1 Beverage item<br>2. Watch the Dashboard and both printers | Within the refresh interval a ticket prints on the Kitchen printer (Chicken Tikka x2) and another on the Beverage printer (Lime Soda x1), each with table name and time. Tickets show names and quantities only, no prices. A green bar says 'Sent to Kitchen, Beverage'. | High | - | Not run |
| KT_02 | No duplicate tickets after refresh | KT_01 completed | 1. Refresh the Dashboard<br>2. Wait 30 seconds | Nothing is printed again for the same order. | High | - | Not run |
| KT_03 | Reprint a ticket | QZ Tray is running on the billing PC; in Dashboard ➜ 🖨️ Printers a printer name is set for Kitchen and Beverage; items have departments assigned | 1. Open an order card on the Dashboard<br>2. Click '🖨️ Reprint KOT' | Department tickets print again marked '** REPRINT **' and a green bar says 'Reprinted to …'. | Medium | - | Not run |
| KT_04 | Printing helper not running | QZ Tray is running on the billing PC; in Dashboard ➜ 🖨️ Printers a printer name is set for Kitchen and Beverage; items have departments assigned | 1. Close QZ Tray<br>2. Customer places an order<br>3. Read the bar on the Dashboard<br>4. Start QZ Tray and wait for the next refresh | A red bar says tickets could not be printed and that QZ Tray is not running. After QZ Tray is started the same tickets print without a new order, and each prints only once. | High | - | Not run |
| KT_05 | Department without a printer name | Kitchen printer set; Beverage printer name left empty | 1. Customer orders 1 Kitchen item and 1 Beverage item | Kitchen ticket prints. A red bar says no printer name is set for Beverage, and the Beverage item is retried on later refreshes until a name is saved. | Medium | - | Not run |
| KT_06 | Item without a department | QZ Tray is running on the billing PC; in Dashboard ➜ 🖨️ Printers a printer name is set for Kitchen and Beverage; items have departments assigned; default department set to Kitchen; one item has no department | 1. Customer orders the item with no department | The ticket prints on the Kitchen (default department) printer. | Medium | - | Not run |
| KT_07 | Auto-print switch | QZ Tray is running on the billing PC; in Dashboard ➜ 🖨️ Printers a printer name is set for Kitchen and Beverage; items have departments assigned | 1. Dashboard ➜ 🖨️ Printers ➜ untick 'Auto-print new orders'<br>2. Customer places an order | No tickets print for the new order. Re-ticking the box restores printing for later orders. | Low | - | Not run |
| KT_08 | Order placed while Dashboard is closed | QZ Tray is running on the billing PC; in Dashboard ➜ 🖨️ Printers a printer name is set for Kitchen and Beverage; items have departments assigned | 1. Close the Dashboard tab<br>2. Customer places an order<br>3. Reopen the Dashboard | The kitchen tickets for the missed order print once the Dashboard is opened. | High | KL-03 | Not run |
| KT_09 | Dashboard open on two devices | QZ Tray is running on the billing PC; in Dashboard ➜ 🖨️ Printers a printer name is set for Kitchen and Beverage; items have departments assigned; Dashboard open on two devices | 1. Customer places an order | Each department ticket prints exactly once. | High | KL-02 | Not run |

</details>

<details>
<summary><b>Reports & Settlement</b> (10)</summary>

| ID | Scenario | Pre-condition | Steps | Expected result | Priority | Ref | Status |
|---|---|---|---|---|---|---|---|
| RP_01 | Settle bill with single payment | Bill saved with pending payment | 1. Open Today's Report<br>2. Click Settle on the bill<br>3. Select 'Cash'<br>4. Confirm | Bill moves from Pending to Settled; daily_reports total orders +1, total revenue and cash revenue increase by bill amount. | High | KL-07 | Not run |
| RP_02 | Split payment amounts mismatch | Pending bill with final amount ₹990 | 1. Click Settle ➜ choose Split (Cash + UPI)<br>2. Enter Cash 500, UPI 400<br>3. Confirm | Error 'Amounts must add up to ₹990 (currently ₹900)'; bill stays pending. With Cash 500 + UPI 490 it settles and both revenues update. | High | - | Not run |
| RP_03 | Close day with pending bills | At least one pending bill today | 1. Click Close Day ➜ confirm | Alert '⚠️ N bills are still unsettled.' and day is not closed. After settling all bills Close Day shows '✅ Day closed!' and bills become 'day_closed'. | High | - | Not run |
| RP_04 | Date-range report accuracy | Several paid bills across 3 days exist | 1. Open Reports<br>2. Select From/To date range (IST)<br>3. Compare totals with sum of individual bills | Revenue, bill count, cash/UPI/card totals equal the sum of bills; split payments counted under each method; 0-quantity items excluded; bills at 23:59 IST fall on the correct day. | High | DEF-09 | Not run |
| RP_05 | Export report to CSV | Report with data displayed | 1. Click the export (CSV) button on a report tab<br>2. Open the downloaded .csv in Excel | A .csv (e.g. DateRange_<from>_to_<to>.csv) downloads with the same rows and totals as the screen, including split-payment text and IST dates; ₹/Rs values and Hindi/special characters display correctly (UTF-8 BOM). | Medium | - | Not run |
| RP_06 | Settle as complimentary | Pending bill: items ₹900, service charge 10% (payable ₹990) | 1. Today's Report ➜ Settle Payment on the bill<br>2. Choose 🎁 Free and press 'Mark Free' with the reason blank<br>3. Enter the reason "Owner's guest" and press 'Mark Free'<br>4. Read the confirmation popup and press OK | Blank reason is blocked ('A reason is required for a complimentary bill'). The popup states items worth ₹900 will be given free and the customer pays ₹0. After OK the bill moves to Settled with a 🎁 Complimentary badge, total ₹0, its items still listed and 'Given free · items worth ₹900'. Bill count increases by 1; revenue does not change. | High | - | Not run |
| RP_07 | Complimentary shown separately in reports | One cash bill ₹500 and one complimentary bill (items ₹900) settled today | 1. Open Reports ➜ Today and note revenue, discounts and average order value<br>2. Open Settlement<br>3. Open Discounts<br>4. Open Table-wise for the complimentary table | Today and Settlement show a Complimentary card (1 bill, items worth ₹900). Revenue, Cash/UPI/Card and Net Collected include only the ₹500 bill. Average order value, service charge and discount % ignore the complimentary bill. It is not listed in Discounts. Table-wise shows its items with a ₹0 bill total. | High | - | Not run |
| RP_08 | Unsettled (pending) bills reported separately | One bill printed from the Dashboard and left unsettled | 1. Open Reports ➜ Settlement<br>2. Compare Cash + UPI + Card with Grand Total | The unsettled bill appears as Pending. Cash + UPI + Card excludes it, and the difference between Grand Total and collected money equals the pending amount. | Medium | - | Not run |
| RP_09 | Same totals on every tab | At least 5 bills today: cash, UPI, card, split, one with discount, one complimentary | 1. Note Grand Total on Today, Date Range (today ➜ today), Settlement and Table-wise<br>2. On Settlement check Gross + Service − Discounts − Complimentary | Grand Total, bill count and payment-mode totals are identical on every tab, and Gross + Service − Discounts − Complimentary equals Grand Total. | High | - | Not run |
| RP_10 | Modify pending bill – discount limits | Pending bill of ₹1000 (no service charge) | 1. Today's Report ➜ Modify<br>2. Enter 100% ➜ observe<br>3. Enter 10% with no reason ➜ Print & Save<br>4. Enter reason ➜ Print & Save | 100% shows 'Percent discount must be between 1 and 99' and Print & Save is disabled. 10% without a reason is blocked. With a reason the bill saves at ₹900 and the receipt prints. | Medium | DEF-08 | Not run |

</details>

<details>
<summary><b>Settings & Admin Code</b> (7)</summary>

| ID | Scenario | Pre-condition | Steps | Expected result | Priority | Ref | Status |
|---|---|---|---|---|---|---|---|
| ST_01 | Save restaurant settings | Admin logged in | 1. Open Settings<br>2. Fill name, address, phone, GST, footer<br>3. Observe Receipt Preview<br>4. Click 'Save Settings'<br>5. Print a bill | Preview updates live; message 'Settings saved! ✅'. Printed receipt shows saved values. | Medium | - | Not run |
| ST_02 | Restaurant name mandatory | Settings page open | 1. Clear Restaurant Name<br>2. Click Save Settings | Alert 'Restaurant name is required!' and nothing saved. | Medium | - | Not run |
| ST_03 | Invalid GST / phone formats | Settings page open | 1. Enter GST 'ABC123' and phone 'abcd'<br>2. Click Save | Invalid GSTIN (must be 15 chars) and non-numeric phone should be rejected with a clear message. | Low | KL-05 | Not run |
| ST_04 | Admin Code gate – wrong and correct password | Logged in as admin; open /admin/admin-code | 1. Enter a wrong code ➜ Unlock<br>2. Enter the correct code ➜ Unlock (or press Enter)<br>3. Click 🔒 Lock<br>4. Run npm run build and search the files in dist/assets for the correct code | Wrong code shows '❌ Wrong password', the box shakes and clears. Correct code opens the bill control panel. Lock returns to the gate. The code is not found anywhere in the built JavaScript. | High | DEF-02 | Not run |
| ST_05 | Edit a paid bill | Admin Code unlocked; paid bill exists | 1. Load bills for today<br>2. Click edit<br>3. Change qty of an item, add a menu item, set service 5%, discount 10%<br>4. Save | Live totals update before saving; after save alert '✅ Bill updated successfully!' and Reports show the new subtotal/final amount, item quantities and date/time. A bill_audit_log entry (who, when, bill before the change) is saved for the edit. | High | DEF-03 | Not run |
| ST_06 | Delete a paid bill | Admin Code unlocked; paid bill exists | 1. Click delete on a bill<br>2. Cancel<br>3. Click delete again and confirm | Cancel keeps bill. Confirm deletes all orders/items for that bill group and it disappears from Reports and totals. A bill_audit_log entry with the deleted bill's details is saved. | High | DEF-03 | Not run |
| ST_07 | Reports title opens Admin Code | Admin logged in; on /admin/reports | 1. Open Reports page<br>2. Click the 'Reports' title in the top bar<br>3. Log out, then repeat steps 1-2 after logging in again | App navigates to /admin/admin-code and shows the 'Admin Code' password gate (not the bill panel). '← Dashboard' button still goes to /admin/dashboard. | Medium | - | Not run |

</details>

## Still not fixed

These are open. I'd rather list them than hide them, and each one has a test case that will pass once it's fixed.

| ID | Problem | Test |
|---|---|---|
| KL-01 | Table PIN verification currently runs in the customer app. Planned: check it on the server. | - |
| KL-02 | The 'already printed' record for kitchen tickets is kept per browser, so two open Dashboards can print duplicates. Planned: store it in the database. | KT_09 |
| KL-03 | Kitchen tickets print only for orders placed while the Dashboard tab is open. Planned: print everything not yet marked as printed. | KT_08 |
| KL-04 | On the admin login page the Enter key does not submit the form. | AL_06 |
| KL-05 | GST number and phone format are not validated in Settings. | ST_03 |
| KL-06 | The 10-table limit is enforced in the interface only. | TM_03 |
| KL-07 | Daily totals are updated by read-then-write, so two settlements at the same instant could lose an update. | RP_01 |

## Running the tests yourself

1. Deploy the app with a Supabase project and run the SQL setup scripts.
2. Make an admin user, a few menu items with categories and departments, and at least three tables.
3. Use two devices: one as the customer, one as the admin.
4. Go module by module, high priority first, and write the result in the Excel sheet.

Files in this folder:

- `TEST_REPORT.md` is this page
- `QR_Menu_App_Test_Cases_Final.xlsx` has the same cases as a sheet with a live summary, for recording results
