# Eloquent IFRS

[![Build Status](https://travis-ci.com/ekmungai/eloquent-ifrs.svg?branch=master)](https://travis-ci.com/ekmungai/eloquent-ifrs)
[![Test Coverage](https://api.codeclimate.com/v1/badges/7afac1253d0f662d1cfd/test_coverage)](https://codeclimate.com/github/ekmungai/eloquent-ifrs/test_coverage)
[![Maintainability](https://api.codeclimate.com/v1/badges/7afac1253d0f662d1cfd/maintainability)](https://codeclimate.com/github/ekmungai/eloquent-ifrs/maintainability)
![PHP 7.2](https://img.shields.io/badge/PHP-7.2-blue.svg)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

This Package enables any Laravel application to generate [International Financial Reporting Standards](https://www.ifrs.org/issued-standards/list-of-standards/conceptual-framework/) compatible Financial Statements by providing a fully featured and configurable Double Entry accounting subsystem.

The package supports multiple Entities (Companies), Account Categorization, Transaction assignment, Start of Year Opening Balances and accounting for VAT Transactions. Transactions are also protected against tampering via direct database changes ensuring the integrity of the Ledger.

The motivation for this package can be found in detail on my blog post [here](https://karanjamungai.com/posts/accounting_software/)
## Table of contents
1. [Installation](#installation)
2. [Configuration](#configuration)
3. [Usage](#usage)
4. [Getting involved](#getting-involved)
5. [Contributing](#contributing)
6. [Roadmap](#roadmap)
7. [License](#license)
8. [References](#references)

## Installation

Use composer to Install the package into your laravel or lumen application. Laravel IFRS requires PHP version 7.2 and Laravel or Lumen version 5.0 and above.

#### For production

```
composer require "ekmungai/eloquent-ifrs"
composer install --no-dev
```

Then run migrations to create the database tables.

```
php artisan migrate
```

#### For development

Clone this repo, and then run Composer in local repo root to pull in dependencies.

```
git clone git@github.com/ekmungai/eloquent-ifrs eloquent-ifrs
cd eloquent-ifrs
composer update
```

To run the tests:

```
cd eloquent-ifrs
vendor/bin/phpunit
```

## Configuration

The package installs with the default settings as regards the names of Accounts/Transactions Types, Report Titles and Section names as well as Accounts Codes. To adjust these settings use the Laravel artisan publish command to install the ifrs configuration to your application's config folder where you can edit it.

```
php artisan vendor:publish
```

## Usage

This simple example covers the four scenarios to demonstrate the use of the package. First, a description of a Cash Sale to a customer, then a Credit Sale (Invoice) to a client, then a Cash Purchase for an operations expense and finally a Credit Purchase (Bill) from a Supplier for a non operations purpose (Asset Purchase).

First we'll setup the Company (Reporting Entity) and required Accounts to record the Transactions. (Assuming that a registered User already exists):

```
use Ekmungai\IFRS\Models\Entity;
use Ekmungai\IFRS\Models\Currency;

$currency = Currency::new("Euro", "EUR")->save(); //Entities require a reporting currency
$entity = Entity::new("Example Company", $currency)->save();

```
We also need the VAT Rates that apply to the Entity:

```
use Ekmungai\IFRS\Models\Vat;

$outputVat = Vat::new(
    "Output VAT",           // VAT Name/Label
    "Standard Output Vat",  // VAT Description
    20                      // VAT Rate
)->save();

$inputVat = Vat::new("Input VAT", "Standard Input Vat", 10)->save();
$zeroVat = Vat::new("Zero VAT", "Vat for Zero Vat Transactions", 0)->save();
```

Now we'll set up some Accounts:

```

use Ekmungai\IFRS\Models\Account;

$bankAccount = Account::new(
    "Bank Account",         // account name
    Account::BANK,          // account type
)->save();

$revenueAccount = Account::new("Sales Account",Account::OPERATING_REVENUE)->save();

$clientAccount = Account::new("Example Client Account", Account::RECEIVABLE)->save();

$supplierAccount = Account::new("Example Supplier Account", Account::Account::PAYABLE)->save();

$opexAccount = Account::new("Operations Expense Account", Account::OPERATING_EXPENSE)->save();

$assetAccount = Account::new("Office Equipment Account", Account::NON_CURRENT_ASSET)->save();

$salesVatAccount = Account::new("Sales VAT Account", Account::CONTROL_ACCOUNT)->save();

$purchasesVatAccount = Account::new("Input VAT Account", Account::CONTROL_ACCOUNT)->save();

```

Now that all Accounts are prepared, we can create the first Transaction, a Cash Sale:

```
use Ekmungai\IFRS\Transactions\CashSale;

$cashSale = CashSale::new(
    $bankAccount,         // Main Account
    Carbon::now(),        // Transaction Date(time)
    "Example Cash Sale"   // Transaction Narration
)->save(); // Intermediate save does not record the transaction in the Ledger

```
So far the Transaction has only one side of the double entry, so we create a Line Item for the other side:

```
use Ekmungai\IFRS\models\LineItem;

$cashSaleLineItem = LineItem::new(
    $revenueAccount,                    // Revenue Account
    $outputVat,                         // Output VAT
    100,                                // Item Price
    1,                                  // Quantity
    "Example Cash Sale Line Item",      // Item Narration
    $salesVatAccount                    // Vat Account
)->save();

$cashSale->addLineItem($cashSaleLineItem);
$cashSale->post(); // This posts the Transaction to the Ledger

```
The rest of the transactions:

```
use Ekmungai\IFRS\Transactions\ClientInvoice;

$clientInvoice = ClientInvoice::new($clientAccount, Carbon::now(), "Example Credit Sale")
//Line Item save may be skipped as saving the Transaction saves the all its Line Items automatically
->addLineItem(
  $LineItem::new($revenueAccount, $outputVat, 50, 2, "Example Credit Sale Line Item", $salesVatAccount)
)
//Transaction save may be skipped as post() saves the Transaction automatically
->post();

use Ekmungai\IFRS\Transactions\CashPurchase;

$cashPurchase = CashPurchase::new($bankAccount, Carbon::now(), "Example Cash Purchase")
->addLineItem(
  LineItem::new($opexAccount, $inputVat, 25, 4, "Example Cash Purchase Line Item", $purchaseVatAccount)
)->post();

use Ekmungai\IFRS\Transactions\SupplierBill;

SupplierBill::new($supplierAccount, Carbon::now(), "Example Credit Purchase")
->addLineItem(
  LineItem::new($assetAccount, $inputVat, 25, 4, "Example Credit Purchase Line Item", $purchaseVatAccount)
)->post();

use Ekmungai\IFRS\Transactions\ClientReceipt;

ClientReceipt::new($clientAccount, Carbon::now(), "Example Client Payment")
->addLineItem(LineItem::new($bankAccount, $zeroVat, 50, 1, "Part payment for Client Invoice")
->post();

```
We can assign the receipt to partially clear the Invoice above:

```
use Ekmungai\IFRS\Models\Assignment;

echo $clientInvoice->clearedAmount(); //0: Currently the Invoice has not been cleared at all
echo $clientReceipt->balance(); //50: The Receipt has not been assigned to clear any transaction

$assignment = Assignment::new(
    $clientReceipt,   // Clearing Transaction
    $clientInvoice,   // Transaction being cleared
    50                // Amount to be cleared
)->save();

echo $clientInvoice->clearedAmount(); //50
echo $clientReceipt->balance(); //0: The Receipt has been assigned fully to the Invoice

```
We have now some Transactions in the Ledger, so lets generate some reports. First though, Reports require a reporting period:

```
use Ekmungai\IFRS\Models\ReportingPeriod;

$period = ReportingPeriod::new(2020)->save();

```
The Income Statement (Profit and Loss):

```
use Ekmungai\IFRS\Reports\IncomeStatement;

$incomeStatement = new IncomeStatement(
    "2020-01-01",   // Report start date
    "2020-12-31",   // Report end date
)->getSections();// Fetch balances from the ledger and store them internally

/**
* this function is only for demonstration and
* debugging use and should never be called in production
*/
dd($incomeStatement->toString());

Example Company
Income Statement
For the Period: Jan 01 2020 to Dec 31 2020

Operating Revenues
    Operating Revenue        200 (100 cash sales + 100 credit sales)

Operating Expenses
    Operating Expense        100 (cash purchase)
                        ---------------
Operations Gross Profit      100

Non Operating Revenues
    Non Operating Revenue    0
                        ---------------
Total Revenue                100

Non Operating Expenses
    Direct Expense           0
    Overhead Expense         0
    Other Expense            0
                        ---------------
Total Expenses               0
                        ---------------
Net Profit                   100
                        ===============

```
The Balance Sheet:

```
use Ekmungai\IFRS\Reports\BalanceSheet;

$balanceSheet = new BalanceSheet(
    "2020-12-31"  // Report end date
)->getSections();

/**
* again to emphasize, this function is only for demonstration and
* debugging use and should never be called in production
*/
dd($balanceSheet->toString());

Example Company
Balance Sheet
As at: Dec 31 2020

Assets
    Non Current Asset        120 (asset purchase)
    Receivables              70  (100 credit sale + 20 VAT - 50 client receipt)
    Bank                     50  (120 cash sale - 120 cash purchase + 50 client receipt)
                        ---------------
Total Assets                 240

Liabilities
    Control Account          20  (VAT: 20 cash sale + 20 credit sale - 10 cash purchase - 10 credit purchase)
    Payable                  120 (100 credit purchase + 20 VAT)
                        ---------------
Total Liabilities            140

                        ---------------
Net Assets                   100
                        ===============

Equity
    Income Statement         100
                        ---------------
Total Equity                 100
                        ===============

```
While the Income Statement and Balance Sheet are the ultimate goal for end year (IFRS) reporting, the package also provides intermediate period reports including Account Statement, which shows a chronological listing of all Transactions posted to an account ending with the current balance for the account; and Account Schedule, which is similar to an Account Statement with the difference that rather than list all Transactions that constitute the ending balance the report only shows the outstanding (Uncleared) Transactions.

In the above example:

```
use Ekmungai\IFRS\Reports\AccountStatement;
use Ekmungai\IFRS\Reports\AccountSchedule;

$statement = new AccountStatement($clientAccount)->getTransactions();

dd($statement->transactions);

array:2[
  ["transaction" => ClientInvoice, "debit" => 120, "credit" => 0, "balance" => 120],
  ["transaction" => ClientReceipt, "debit" => 0, "credit" => 50, "balance" => 70]
]

$schedule = new AccountSchedule($clientAccount, $currency)->getTransactions();

dd($schedule->transactions);

array:1[
  ["transaction" => ClientInvoice, "amount" => 120, "cleared" => 50, "balance" => 70],
]

```
## Getting Involved

I am acutely aware that as a professionally trained Accountant I may have used some conventions, definitions and styles that while seemingly obvious to me, might not be so clear to another developer. I would therefore welcome and greatly appreciate any feedback on the ease of use of the package so I can make it more useful to as many people as possible.


## Contributing

1. Fork it (<https://github.com/ekmungai/eloquent-ifrs/fork>)
2. Create your feature branch (`git checkout -b feature/fooBar`)
3. Write tests for the feature
3. Commit your changes (`git commit -am 'Add some fooBar'`)
4. Push to the branch (`git push origin feature/fooBar`)
5. Create a new Pull Request

## Roadmap

* Complete Documentation
* Add Multicurrency support
* Add Receivables(Debtors)/Payables(Creditors) Aging Balances analysis Report
* Add Cashflow Statement
* Add Changes in Equity Statement


## License
This software is distributed for free under the MIT License


## References
This package is heavily influenced by [chippyash/simple-accounts-3](https://github.com/chippyash/simple-accounts-3) and [scottlaurent/accounting](https://github.com/scottlaurent/accounting).