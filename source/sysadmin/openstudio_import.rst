OpenStudio import
=============================

This page will explain how to import your OpenStudio data into Costasiella.
Please note that the import might override some information like details of your organization. 
It's therefore strongly recommended to create a database & file backup of your Costasiella installation before proceeding any further.

Manual import
-----------------------

A few items can't be migrated automatically and should therefore be migrated manually.

- Settings
- Mail templates
- Events without a school location set
- Event ticket earlybird discounts
- School subscription price glaccount and costcenter information
- Prices for drop-in classes & trial classes
- Groups and group permissions

Data imported
--------------

**Organization (school)**

- sys_organizations > organization
- accounting_glaccounts > finance_glaccount
- accounting_costcenters > finance_costcenter
- tax_rates > finance_tax_rate
- payment_methods > finance_payment_method
- school_memberships > organization_membership
- school_classcards > organization_classpass
- school_classcards_groups > organization_classpass_group
- school_classcards_groups_classcard > organization_classpass_group_classpass
- school_classtypes > organization_classtype
- school_discovery > organization_discovery
- school_levels > organization_level
- school_locations > organization_location (Add one room for each added location)
- school_shifts > organization_shifts
- school_subscriptions > organization_subscription
- school_subscriptions_groups > organization_subscription_group
- school_subscriptions_groups_subscriptions > organization_subscription_group_subscription
- school_subscriptions_price > organization_subscription_price

**Customer accounts**

- auth_user (customers) > account & all_auth_email
- auth_user (businesses) > business
- auth_user (teacher info) > account_instructor_profile
- customers_classcards > account_classcard 
- customers_notes > account_note
- customers_orders > finance_order
- customers_orders_amounts > finance_order
- customers_orders_items > finance_order_item
- customers_orders_mollie_payment_ids > integration_log_mollie
- customers_payment_info > account_bank_account 
- customers_payment_info_mandates > account_bank_account_mandate
- customers_subscriptions > account_subscription
- customers_subscriptions_alt_prices > account_subscription_alt_price
- customers_subscriptions_blocked > account_subscription_block
- customers_subscriptions_credits > account_subscription_credit 
- customers_subscriptions_paused > account_subscription_pause

**Invoices**

- invoices_groups > finance_invoice_group
- invoices_groups_product_types > finance_invoice_group_default
- invoices > finance_invoice
- invoices_amounts > finance_invoice
- invoices_customers > finance_invoice
- invoices_items > finance_invoice_item
- invoices_items_customers_classcards > finance_invoice_item
- invoices_items_customers_subscriptions > finance_invoice_item
- invoices_items_workshop_products_customers > finance_invoice_item
- invoices_payments > finance_invoice_payment
- invoices_mollie_payment_ids > integration_log_mollie

**Payment batches**

- payment_batches > finance_payment_batch
- payment_batches_export > finance_payment_batch_export
- payment_batches_items > finance_payment_batch_item
- payment_categories > finance_payment_batch_category
- alternativepayments > account_finance_payment_batch_category_item

**Class schedule**

- classes > schedule_item
- classes_mail > schedule_item
- classes_attendance > schedule_item_attendance
- invoices_items_classes_attendance > schedule_item_attendance
- classes_otc > schedule_item_weekly_otc
- classes_otc_mail > schedule_item_weekly_otc
- classes_school_classcards_groups > schedule_item_organization_classpass_group
- classes_school_subscriptions_groups > schedule_item_organization_subscription_group
- classes_teachers > schedule_item_account

**Staff schedule**

- shifts > schedule_item
- shifts_otc > schedule_item_weekly_otc
- shifts_staff > schedule_item_account

**Events / workshops**

- workshops > schedule_event
- workshops > schedule_event_media 
- workshops_mail > schedule_event
- workshops_activities > schedule_item
- workshop_activities_customers > schedule_item_attendance
- workshops_products > schedule_event_ticket
- workshops_products_activities > schedule_event_ticket_schedule_item
- workshops_products_customers > account_schedule_event_ticket

**Announcements**

- announcements > organization_announcements
- customers_profile_announcements > organization_announcements

Import data
------------

OpenStudio data can be imported using the *openstudio_import* management command.

.. code-block:: bash
    
    ./manage.py openstudio_import --db_name=<openstudio> --db_user=<user> --db_password=<password> --db_host=<openstudio db server> --os_uploads_folder=<path/to/web2py/applications/openstudio/uploads>

Review import log
------------------

After the import a new log file containing import errors (if any) will be available in the logs directory in the Costasiella application root folder.
