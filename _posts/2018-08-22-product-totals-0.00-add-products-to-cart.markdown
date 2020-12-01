---
layout: post
title:  "Product totals calculating to $0.00 when adding products to cart programmatically"
date:   2018-08-22 22:33:00 -0500
categories: magento-2
permalink: /magento-2/product-totals-calculating-to-0-00-when-adding-products-to-cart-programmatically
---
I had this problem recently where the cart page was displaying $0.00 on all the product line items after I had programmatically added 3 products to my cart. I didn't find a solution after a quick Google search, so after lots of `CMD+B` in PHPStorm and reading the `checkout/cart/add` controller, I had my solution. The block of code below is a re-creation of what I was doing to add these products to my cart.

{% highlight php %}
/**
 * @var \Magento\Checkout\Model\Cart
 */
protected $customerCart;

/**
 * @var \Magento\Catalog\Model\ProductFactory
 */
protected $productFactory;

public function execute()
{
    $productIds = ['1', '2', '3'];

    $productCollection = $this->productFactory->create()
        ->getCollection()
        ->addFieldToFilter('entity_id', ['in' => $productIds])
        ->addWebsiteFilter($this->storeManager->getWebsite()->getId());

    foreach ($productCollection as $product) {
        $this->customerCart->addProduct($product, [
            'qty' => 1
        ]);
    }

    $this->customerCart->save();
}
{% endhighlight %}

`->addAttributeToSelect('*');` to the collection, and my totals were calculating correctly again.

`$this->customerCart->save()`, the quote begins `$quote->collectTotals()`. Because the `price` attribute was never loaded when I executed the collection, when `$product->getData('price')` returns `NULL` and ends up calculating all of the totals to $0.00.

You may ask "why select all attributes instead of just the price attribute?" I noticed that in the `/checkout/cart/add` controller, the product is loaded via the `Magento\Catalog\Model\ProductRepository`, which also loads all product attributes. Any third party module can also inject their own totals collector, and you never know what attributes they could need during calculations.