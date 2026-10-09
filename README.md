# Web-Tables

```
Use the same demo website and complete these tasks.

https://assertqa.com/practice/webtables

Test case

Task

Expected result

TC01

Print all column headings

All headings displayed

TC02

Print the first data row

First employee record displayed

TC03

Print the last data row

Last employee record displayed

TC04

Search for an employee by last name

Matching record displayed

TC05

Extract all email addresses

Every email printed

TC06

Find the employee with the highest Due amount

Matching employee identified

TC07

Verify that a particular website link exists

Pass or fail reported

TC08

Count data rows without counting the header

 ```


# PROGRAM

```
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import Select, WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import re
from urllib.parse import urlparse

driver = webdriver.Chrome()
wait = WebDriverWait(driver, 15)

try:
    # Open the required demo website
    driver.maximize_window()
    driver.get("https://assertqa.com/practice/webtables")

    # Wait for the table to load
    wait.until(
        EC.presence_of_element_located(
            (By.XPATH, "//table")
        )
    )

    table = driver.find_element(By.XPATH, "//table")

    rows_per_page_controls = driver.find_elements(
        By.XPATH, "//select[@aria-label='Rows per page']"
    )
    if rows_per_page_controls:
        rows_per_page_select = Select(rows_per_page_controls[0])
        page_sizes = [
            int(option.get_attribute("value"))
            for option in rows_per_page_select.options
            if (option.get_attribute("value") or "").isdigit()
        ]

        if page_sizes:
            result_summary = driver.find_element(By.TAG_NAME, "body").text
            result_match = re.search(
                r"Showing\s+\d+\s+of\s+([\d,]+)\s+results", result_summary
            )
            if result_match is None:
                raise RuntimeError(
                    "Could not determine the total number of table results"
                )

            total_results = int(result_match.group(1).replace(",", ""))
            page_size = max(page_sizes)
            rows_per_page_select.select_by_value(str(page_size))
            wait.until(
                lambda current_driver: len(
                    current_driver.find_elements(
                        By.XPATH, "//table//tbody/tr"
                    )
                )
                == min(total_results, page_size)
            )

    # Get column headings and data rows
    headers = table.find_elements(By.XPATH, ".//thead//th")
    rows = table.find_elements(By.XPATH, ".//tbody/tr")

    # Fallback if the website does not use thead/tbody
    if not headers:
        headers = table.find_elements(By.XPATH, ".//tr[1]/th")

    if not rows:
        rows = table.find_elements(By.XPATH, ".//tr[td]")

    header_names = [h.text.strip() for h in headers]

    # Find column indexes using their names
    def get_column_index(name):
        for i, heading in enumerate(header_names):
            if name.lower() in heading.lower():
                return i
        return -1

    last_name_col = get_column_index("Last Name")
    email_col = get_column_index("Email")
    due_col = get_column_index("Due")
    website_col = get_column_index("Web Site")

    # TC01: Print all column headings
    print("\nTC01 - ALL COLUMN HEADINGS")
    for heading in header_names:
        print(heading)

    # TC02: Print the first data row
    print("\nTC02 - FIRST DATA ROW")
    if rows:
        print(rows[0].text)
    else:
        print("No data rows found")

    # TC03: Print the last data row
    print("\nTC03 - LAST DATA ROW")
    if rows:
        print(rows[-1].text)
    else:
        print("No data rows found")

    # TC04: Search for an employee by last name
    print("\nTC04 - SEARCH BY LAST NAME")
    search_name = "Smith"  # Change this value as needed
    found = False

    if last_name_col != -1:
        for row in rows:
            cells = row.find_elements(By.XPATH, "./td")

            if (
                len(cells) > last_name_col
                and cells[last_name_col].text.strip().lower()
                == search_name.lower()
            ):
                print("PASS - Employee found")
                print(row.text)
                found = True
                break

    if not found:
        print("FAIL - Employee not found")

    # TC05: Extract all email addresses
    print("\nTC05 - ALL EMAIL ADDRESSES")
    emails = []

    if email_col != -1:
        for row in rows:
            cells = row.find_elements(By.XPATH, "./td")
            if len(cells) > email_col:
                email = cells[email_col].text.strip()
                if email:
                    emails.append(email)

    for email in emails:
        print(email)

    print("Total emails:", len(emails))

    # TC06: Find the employee with the highest Due or Salary amount
    print("\nTC06 - HIGHEST DUE / SALARY AMOUNT")
    highest_amount = None
    highest_employee = None
    amount_col = due_col if due_col != -1 else get_column_index("Salary")

    if amount_col != -1:
        for row in rows:
            cells = row.find_elements(By.XPATH, "./td")

            if len(cells) > amount_col:
                amount_text = cells[amount_col].text.strip()
                amount = re.sub(r"[^0-9.-]", "", amount_text)

                try:
                    amount = float(amount)
                except ValueError:
                    continue

                if (
                    highest_amount is None
                    or amount > highest_amount
                ):
                    highest_amount = amount
                    highest_employee = row.text

    if highest_employee is not None:
        amount_heading = header_names[amount_col]
        print("Employee:", highest_employee)
        print(f"Highest {amount_heading}: ${highest_amount:,.2f}")
    else:
        print("FAIL - Neither a Due nor Salary column with valid amounts was found")

    # TC07: Verify that a website link exists
    print("\nTC07 - WEBSITE LINK VERIFICATION")
    link_found = False

    if website_col != -1:
        for row in rows:
            cells = row.find_elements(By.XPATH, "./td")

            if len(cells) > website_col:
                website_cell = cells[website_col]

                links = website_cell.find_elements(
                    By.XPATH, ".//a[@href]"
                )

                website_text = website_cell.text.strip()

                if links or website_text.startswith(
                    ("http://", "https://")
                ):
                    print("PASS - Website link exists in the table")
                    print(
                        links[0].get_attribute("href")
                        if links
                        else website_text
                    )
                    link_found = True
                    break

    if not link_found:
        current_host = urlparse(driver.current_url).netloc.lower()

        for link in driver.find_elements(By.XPATH, "//a[@href]"):
            href = link.get_attribute("href")
            parsed_href = urlparse(href or "")

            if parsed_href.netloc and parsed_href.netloc.lower() != current_host:
                print(
                    "PASS - External website link exists on the page "
                    "(no Website column in the table)"
                )
                print(href)
                link_found = True
                break

    if not link_found:
        print("FAIL - No website link found in the table or on the page")

    # TC08: Count data rows without the header
    print("\nTC08 - DATA ROW COUNT")
    print("Visible data rows:", len(rows))
    print("Number of columns:", len(header_names))

    finally:
    driver.quit()
```

<img width="1916" height="1016" alt="Screenshot 2026-10-09 141741" src="https://github.com/user-attachments/assets/85e4c5a7-b9d3-497d-8d7c-da5e8ff99f6d" />

