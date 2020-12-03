---
layout: post
title:  "Custom Event Sourcing"
date:   2020-12-02 23:39:00 -0500
categories: ruby rails event sourcing
interest: app
---
I seem to be late to the Event Sourcing architecture party, like several years late. I initially looked into it sometime ago, but didn't have any immediate application use-case for it, so filed it away until recently when building an enterprise Rails application that tracks daily production activities, and also income and expense (Accounts, yo!).
 
### Enter Event Sourcing...
In its basic form, Event Sourcing revolves around aggregates -- a body of interconnected entities (models) which together complete a task. The classic example is that of an Order: select order items, make payment, order shipped, order delivered. These activities (or events) leave a trail of interesting data points both at the merchant and buyer ends. The aggregate here is composed of `:order`, `:payment`, `:shipment` and `:item` entities, with `:order` serving as aggregate_root

### Implementing Event Sourcing in Rails
There are a few ideas to implementing Event Source I felt were worthy of trying out. These include setting up `rails_event_store` gem and CommandBus with Redis, and using ActiveRecord::Observer to track model changes. But after reading a bunch of articles on the subject, I thought to bite the bullet, and roll my own implementation, while drawing inspiration from [Kickstarter's guide][kickstarter-guide] -- their barebones no frills approach offers me customization opportunities down the line.

### What We Will be Building
Our example will be based on an Account Reconciliation app that keeps track of a small business' income and expenses. These (Account, Income, and Expenses) form an Aggregate cluster with Account as Aggregate Root, and Income & Expense serving as events. When a business receives or pays out money, we would expect it to impact the Accounts. 

### The Solution Design
With my (admittedly limited) understanding of Event Sourcing, I decided to model the solution as a Multi-Table Inheritance, and got some help from [here][multi-table-inheritance]. This gave me some quick wins, and I was able to get my model running with this setup:
```
    class Account < ApplicationRecord    
       # attributes :amount, :balance, :data, :status    
       enum status: { pending_approval: 0, paid: 1, deposited: 2, failed: 3 }    

       before_validation :initialize_model    

       validates :amount, presence: true,  numericality: { greater_than_or_equal_to: 0 }

       # to be implemented in subclasses that are part of the Account aggregate
       def initialize_model
           raise NotImplementedError, 'Subclass did not define #initialize_model'
       end

    end
```
```
    class Account::Salary < Account    
        has_one :event, class_name: 'SalaryEvent', inverse_of: :salary, dependent: :destroy, autosave: true
    
        delegate :basic, :basic=, :allowances, :allowances=, :tax_payable, :tax_payable=, :prepared_by, :prepared_by=, :approved_by, :approved_by=, to: :lazily_built_event
    
        validates :basic, :allowances, :tax_payable, presence: true, numericality: { only_integer: true, greater_than: 0 }
    
        def lazily_built_event        
            event || build_event   
        end

        # implement the inherited method. This doubles as our Calculator    
        def initialize_model        
            amount = total_cost        
            data = { basic: basic, allowances: allowances, tax_payable: tax_payable, prepared_by: prepared_by, approved_by: approved_by }.as_json            
            
            if approved_by            
                balance = get_balance - total_cost
                status = 'paid'         
            else            
                status = 'pending_approval'  
            end    
        end

        private

        def total_cost
            basic + allowances + tax_payable
        end

        def get_balance
            (balance || 0)
        end

    end
```

```
    class SalaryEvent < ApplicationRecord    
        # attributes :basic, :allowances, :tax_payable, :prepared_by, approved_by  
        belongs_to :salary, class_name: 'Account::Salary', inverse_of: :event

        validates :salary, presence: true
    end
```
With this implementation, I was able to get AggregateRoot, Events, and Calculator functional. Next up: `Reactors & Dispatch`

### Conclusion
This is a simplistic Account tracking application. The goal is to show some nuances with Event Sourcing that warrants pecuniar treatment (e,g. Double Entry and Dual Control principles in Accounting)

See you next time!


[kickstarter-guide]: https://kickstarter.engineering/event-sourcing-made-simple-4a2625113224
[multi-table-inheritance]: https://belighted.com/blog/implementing-multiple-table-inheritance-in-rails
