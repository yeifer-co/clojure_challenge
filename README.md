# Clojure Engineering Challenge

> Learning project for clojure

> All commits on this repository should follow the [commit convention](doc/commit_convention.md).

## Getting started

This clojure challenge is made up of 3 questions that reflect the learning you accumulated for the past week. Complete the following instructions:

1. Create a Github/Gitlab repo to show the challenge code. When complete, send us the link to your challenge results.
2. Duration: About 4-6 hours
3. Install Cursive Plugin to Intellij and setup a clojure deps project. https://cursive-ide.com/userguide/deps.html
4. Enjoy!

> Instructions for installing InteliJ available in [Install InteliJ](doc/install_InteliJ.md).

## Problems
### Problem 1 Thread-last Operator ->>
Given the invoice defined in **invoice.edn** in this repo, use the thread-last ->> operator to find all invoice items that satisfy the given conditions. Please write a function that receives an invoice as an argument and returns all items that satisfy the conditions described below.
#### Requirements
- Load invoice to play around with the function like this:

```
(def invoice (clojure.edn/read-string (slurp "invoice.edn")))
```

#### Definitions
- An invoice item is a clojure map { … } which has an :invoice-item/id field. EG.

```
{:invoice-item/id     "ii2"  
  :invoice-item/sku "SKU 2"}
```

- An invoice has two fields :invoice/id (its identifier) and :invoice/items a vector of invoice items

#### Invoice Item Conditions
- At least have one item that has :iva 19%
- At least one item has retention :ret\_fuente 1%
- Every item must satisfy EXACTLY one of the above two conditions. This means that an item cannot have BOTH :iva 19% and retention :ret\_fuente 1%.

#### Solution

The solution is a clojure script. It is available in the src folder of this repository.
[Click here to see script](src/invoice_play_around.clj).

```clojure
;^; Problem 1 Solution
(ns invoice_play_around
  (:require [clojure.edn :as edn]))

; Load invoice to play around with it
(def invoice (edn/read-string (slurp "invoice.edn")))

; Condition 1: At least have one item that has :iva 19%
(defn condition-1?
  "At least have one item that has :iva 19%"
  [item]
  (some #(= 19 (:tax/rate %)) (:taxable/taxes item)))

; Condition 2: At least one item has retention :ret_fuente 1%
(defn condition-2?
  "At least one item has retention :ret_fuente 1%"
  [item]
  (some #(= 1 (:retention/rate %)) (:retentionable/retentions item)))
(def not-condition-2?
  "No item has retention :ret_fuente 1%"
  (complement condition-2?))

(defn check-conditions
  "Check if an item satisfies the conditions"
  [item]
  ; Every item must satisfy EXACTLY one of the above two conditions.
  (if (condition-1? item)
    (not-condition-2? item)
    (condition-2? item)))

; Filter invoice items by conditions
(defn filter-by-conditions
  "Filter invoice items by conditions"
  [invoice]
  ; Use ->> to thread the invoice items through the filter
  (->> invoice ; thread operator allows to pass the result of the previous expression as the last argument of the next expression
       :invoice/items
       (filter check-conditions)
       (vec)))

(def result (filter-by-conditions invoice))

(defn -main
  "Use this to play around with the invoice"
  [& args]
  ; Print the result
  (println "Original invoice:")
  (println invoice)
  (println "Filtered invoice:")
  (println result))
```

#### Comments

- It was necessary to add `main` function to be able to run the code using "play" button in IntelliJ.
- I tried to apply good functional programming practices, like using pure functions and avoiding side effects.
- It was interesting to learn about the `->>` operator. It is very useful to avoid nested expressions.

## Problem 2: Core Generating Functions
Given the invoice defined in **invoice.json** found in this repo, generate an invoice that passes the spec **::invoice** defined in **invoice-spec.clj**. Write a function that as an argument receives a file name (a JSON file name in this case) and returns a clojure map such that

```
(s/valid? ::invoice invoice) => true 
```

where invoice represents an invoice constructed from the JSON.

#### Solution

The solution is at the end of the `invoice_spec.clj` file. It is a clojure script. It is available in the src folder of this repository.
[Click here to see script](src/invoice_spec.clj).

```clojure
;^; Problem 2 Solution
(defn parse-date
  "Parse a date string into a date object"
  [date-string]
  (let [date-formatter (SimpleDateFormat. "dd/MM/yyyy")]
    (.parse date-formatter date-string)))

(defn read-json-file
  "Read a JSON file and return a map"
  [file-name]
  (json/read-str (slurp file-name) :key-fn keyword))

(defn get-issue-date
  "Get the issue date from the invoice"
  [invoice]
  (parse-date (get-in invoice [:invoice :issue_date])))

(defn get-customer
  "Get the customer from the invoice"
  [invoice]
  (let [customer (get-in invoice [:invoice :customer])]
    {:customer/name  (get-in customer [:company_name])
     :customer/email (get-in customer [:email])}))

(defn get-items
  "Get the items from the invoice"
  [invoice]
  (let [items (get-in invoice [:invoice :items])]
    (map (fn [item]
           {:invoice-item/price    (get-in item [:price])
            :invoice-item/quantity (get-in item [:quantity])
            :invoice-item/sku      (get-in item [:sku])
            :invoice-item/taxes    (vec (map (fn [tax]
                                               {
                                                :tax/category (keyword (clojure.string/lower-case (get-in tax [:tax_category])))
                                                :tax/rate     (double (get-in tax [:tax_rate]))
                                                })
                                             (get-in item [:taxes])))
            })
         items)))

(defn generate-invoice
  "Generate an invoice that passes the corresponding spec"
  [file-name]
  (let [invoice (read-json-file file-name)]
    {:invoice/issue-date (get-issue-date invoice)
     :invoice/customer  (get-customer invoice)
     :invoice/items     (vec (get-items invoice))}))

(defn -main
  "Use this execute the invoice spec"
  [& args]
  (let [invoice (generate-invoice "invoice.json")]
    (println (s/valid? ::invoice invoice))
    (s/explain ::invoice invoice)))
```

#### Comments

- My first approach was to use `clojure.spec.alpha/keys` to define the invoice spec. However, I found it myself in a bottleneck when defining the spec for the nested maps. Looking for a solution, I found a blog post that suggested to use `clojure.spec.alpha/and` and `clojure.spec.alpha/or` to define the spec for the nested maps. However, I was not able to make it work.
- I decide to use plain clojure maps to define the spec. It was easier to define the spec for the nested maps using native functions. However, I am not sure if this is the best approach.
- It was really helpful to use `clojure.spec.alpha/explain` to debug the spec definition.

## Problem 3: Test Driven Development
Given the function **subtotal** defined in **invoice-item.clj** in this repo, write at least five tests using clojure core **deftest** that demonstrates its correctness. This subtotal function calculates the subtotal of an invoice-item taking a discount-rate into account. Make sure the tests cover as many edge cases as you can!
