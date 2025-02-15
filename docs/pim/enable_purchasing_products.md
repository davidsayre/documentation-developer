---
description: Ensure your product catalog is ready for use with full configuration of products that enables purchasing them in the frontend shop.
---

# Enable purchasing products

To enable purchasing from the catalog, the following configuration is required:

- at least [one region and one currency for the shop](#region-and-currency)
- [VAT rates for the product type](#vat-rates)
- at least [one price for the product](#product-price)
- [availability with positive or infinite stock for the product](#product-availability)

!!! tip "Product completeness"

    You can track required tasks related to product configuration in product view's **Completeness** tab.

    ![Product completeness](product_completeness.png)

    This page shows which tasks are finished for the given product, and which are still awaiting completion.

    Click the edit button next to an unfinished task to move directly to the screen where you can add the missing information.

[[% include 'snippets/catalog_permissions_note.md' %]]

## Region and currency

All currencies available in the system must be enabled in the Back Office under **Product Catalog** -> **Currencies**.

Additionally, you must configure currencies valid for specific SiteAccesses
under the `ibexa.system.<scope>.product_catalog.currencies` [configuration key](configuration.md#configuration-files):

``` yaml
ibexa:
    system:
        default:
            product_catalog:
                currencies:
                    - EUR
                    - GBP
                    - PLN
                regions:
                    - germany
                    - uk
                    - poland
```

In the `ibexa_storefront.yaml` file, under the `ibexa.system.<scope>.product_catalog.regions` configuration key, regions are set with `default` value. Remember to either exclude this element or extend it by [configuring other regions](enable_purchasing_products.md#configuring-other-regions-and-currencies).

```yaml
ibexa:
    system:
        storefront_group:
            product_catalog:
                currencies:
                    - EUR
                    - PLN
                regions:
                    - germany
                    - poland 
        another_storefront_group:
            product_catalog:
                currencies:
                    - GBP
                regions:
                    - uk
```

This example uses the currencies and regions set in the [VAT rates' example below](#vat-rates).

### Configuring other regions and currencies

By default, the system always uses the first currency and the first region configured.

To implement a different logic, for example a switcher for preferred currencies and regions,
you need to subscribe to `Ibexa\Contracts\ProductCatalog\Events\CurrencyResolveEvent`
and `Ibexa\Contracts\ProductCatalog\Events\RegionResolveEvent` in your customization.

## VAT rates

You set up VAT percentage values corresponding to VAT rates in configuration:

``` yaml
ibexa:
    repositories:
        default:
            product_catalog:
                engine: default
                regions:
                    germany:
                        vat_categories:
                            standard: 19
                            reduced: 7
                            none: ~
                    uk:                                               
                        vat_categories:                            
                            standard: 20                            
                            reduced: 5                            
                            none: ~
                    poland:                                               
                        vat_categories:                            
                            standard: 23                            
                            reduced: 8                            
                            none: ~
```

You can configure which VAT rate to use per product type in the Back Office.

In the product type editing mode, access the definition of the Product Specification Field.

![Setting up VAT rates for product type](catalog_vat_rates.png)

## Product price

The product must have at least one [price](prices.md) configured.

## Product availability

To enable adding a product to cart, you must configure product [availability](products.md#product-availability-and-stock)
with positive or infinite stock.
