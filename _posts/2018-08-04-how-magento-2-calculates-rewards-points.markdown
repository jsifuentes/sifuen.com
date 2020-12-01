---
layout: post
title:  "How Magento 2 calculates totals with reward points"
date:   2018-08-04 16:28:00 -0500
categories: magento-2
permalink: magento-2/how-magento-2-calculates-totals-with-reward-points
---
<em>* Note: Magento Rewards is an Enterprise module. Community Magento installations will not contain this module.</em>

I recently had to work on a new feature for a client that added the ability for a user to select how many reward points they want to use on their order. By default, at least in Magento 2.1.9, if a customer decides to redeem their reward points during checkout, they are forced to use their entire balance or at least enough to cover the grand total. For this client, they wanted to allow their customers to use any arbitrary amount of reward points on their order and pay the rest using another available payment method.

For this, I needed to figure out how Magento 2 calculates how many reward points to use when a customer presses that button confirming they want to redeem their points. When a customer presses that button, an API call is fired to `/rest/V1/reward/mine/use-reward` which gets routed to `Magento\Reward\Api\RewardManagementInterface::set` which, according to di.xml, is aliased to `Magento\Reward\Model\RewardManagement::set`.

{% highlight php %}
/**
 * @var \Magento\Reward\Model\PaymentDataImporter
 */
protected $importer;

/**
 * {@inheritdoc}
 */
public function set($cartId)
{
    if ($this->rewardData->isEnabledOnFront()) {
        /* @var $quote \Magento\Quote\Model\Quote */
        $quote = $this->quoteRepository->getActive($cartId);
        $this->importer->import($quote, $quote->getPayment(), true);
        $quote->collectTotals();
        $this->quoteRepository->save($quote);
        return true;
    }
    return false;
}
{% endhighlight %}

This function delegates the heavy lifting of setting data on the Quote model to an "importer" called `Magento\Reward\Model\PaymentDataImporter`, followed by a call to `$quote->collectTotals();` to actually calculate the grand total of the order. Following how the code would execute, let's dive into the `Magento\Reward\Model\PaymentDataImporter::import` function.

{% highlight php %}
/**
 * Prepare and set to quote reward balance instance,
 * set zero subtotal checkout payment if need
 *
 * @param \Magento\Quote\Model\Quote $quote
 * @param \Magento\Framework\DataObject $payment
 * @param bool $useRewardPoints
 * @return $this
 */
public function import($quote, $payment, $useRewardPoints)
{
    if (!$quote ||
        !$quote->getCustomerId() ||
        $quote->getBaseGrandTotal() + $quote->getBaseRewardCurrencyAmount() <= 0
    ) {
        return $this;
    }
    $quote->setUseRewardPoints((bool)$useRewardPoints);
    if ($quote->getUseRewardPoints()) {
        $customer = $quote->getCustomer();
        /* @var $reward \Magento\Reward\Model\Reward */
        $reward = $this->_rewardFactory->create()->setCustomer($customer);
        $reward->setWebsiteId($quote->getStore()->getWebsiteId());
        $reward->loadByCustomer();
        $minPointsBalance = (int)$this->_scopeConfig->getValue(
            \Magento\Reward\Model\Reward::XML_PATH_MIN_POINTS_BALANCE,
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE,
            $quote->getStoreId()
        );

        if ($reward->getId() && $reward->getPointsBalance() >= $minPointsBalance) {
            $quote->setRewardInstance($reward);
            if (!$payment->getMethod()) {
                $payment->setMethod('free');
            }
        } else {
            $quote->setUseRewardPoints(false);
        }
    }
    return $this;
}
{% endhighlight %}

So let's run through what this does. The first step is to verify the quote is a real quote, verify the customer is logged in, and verify the quote's total is more than 0. Afterwards, the `use_reward_points` field on the Quote model is set to true (since the call to this function hard codes `$useRewardPoints` to true). The next block of code loads the customer's reward points record and verifies the customer has the minimum required amount of points needed to proceed to checkout. Once all has been validated, the `reward_instance` field on the Quote model is set to an instance of `Magento\Reward\Model\Reward`.

Going back to the `Magento\Reward\Model\RewardManagement::set` function, the next line executes `$quote->collectTotals();` Obviously, the `collectTotals()` function will not contain references to rewards, since they are located in different modules -- the Magento_Quote module can still run perfectly without the Magento_Reward module. Enter the magic of `etc/sales.xml`

In `module-reward/etc/sales.xml`, a new quote totals calculator is set up to execute at `Magento\Reward\Model\Total\Quote\Reward`.

{% highlight xml %}
<section name="quote">
    <group name="totals">
        <item name="reward" instance="Magento\Reward\Model\Total\Quote\Reward" sort_order="1000">
            <renderer name="frontend" instance="Magento\Reward\Block\Checkout\Total"/>
        </item>
    </group>
</section>
{% endhighlight %}

Let's open up `Magento\Reward\Model\Total\Quote\Reward`.

{% highlight php %}
/**
 * Collect reward totals
 *
 * @param \Magento\Quote\Model\Quote $quote
 * @param ShippingAssignmentInterface $shippingAssignment
 * @param Address\Total $total
 * @return $this
 * @SuppressWarnings(PHPMD.CyclomaticComplexity)
 * @SuppressWarnings(PHPMD.UnusedFormalParameter)
 */
public function collect(
    \Magento\Quote\Model\Quote $quote,
    \Magento\Quote\Api\Data\ShippingAssignmentInterface $shippingAssignment,
    \Magento\Quote\Model\Quote\Address\Total $total
) {
    if (!$this->_rewardData->isEnabledOnFront($quote->getStore()->getWebsiteId())) {
        return $this;
    }

    $total->setRewardPointsBalance(0)->setRewardCurrencyAmount(0)->setBaseRewardCurrencyAmount(0);

    if ($total->getBaseGrandTotal() >= 0 && $quote->getCustomer()->getId() && $quote->getUseRewardPoints()) {
        /* @var $reward \Magento\Reward\Model\Reward */
        $reward = $quote->getRewardInstance();
        if (!$reward || !$reward->getId()) {
            $customer = $quote->getCustomer();
            $reward = $this->_rewardFactory->create()->setCustomer($customer);
            $reward->setCustomerId($quote->getCustomer()->getId());
            $reward->setWebsiteId($quote->getStore()->getWebsiteId());
            $reward->loadByCustomer();
        }
        $pointsLeft = $reward->getPointsBalance() - $quote->getRewardPointsBalance();
        $rewardCurrencyAmountLeft = $this->priceCurrency->convert(
            $reward->getCurrencyAmount(),
            $quote->getStore()
        ) - $quote->getRewardCurrencyAmount();
        $baseRewardCurrencyAmountLeft = $reward->getCurrencyAmount() - $quote->getBaseRewardCurrencyAmount();
        if ($baseRewardCurrencyAmountLeft >= $total->getBaseGrandTotal()) {
            $pointsBalanceUsed = $reward->getPointsEquivalent($total->getBaseGrandTotal());
            $pointsCurrencyAmountUsed = $total->getGrandTotal();
            $basePointsCurrencyAmountUsed = $total->getBaseGrandTotal();

            $total->setGrandTotal(0);
            $total->setBaseGrandTotal(0);
        } else {
            $pointsBalanceUsed = $reward->getPointsEquivalent($baseRewardCurrencyAmountLeft);
            if ($pointsBalanceUsed > $pointsLeft) {
                $pointsBalanceUsed = $pointsLeft;
            }
            $pointsCurrencyAmountUsed = $rewardCurrencyAmountLeft;
            $basePointsCurrencyAmountUsed = $baseRewardCurrencyAmountLeft;

            $total->setGrandTotal($total->getGrandTotal() - $pointsCurrencyAmountUsed);
            $total->setBaseGrandTotal($total->getBaseGrandTotal() - $basePointsCurrencyAmountUsed);
        }
        $quote->setRewardPointsBalance($quote->getRewardPointsBalance() + $pointsBalanceUsed);
        $quote->setRewardCurrencyAmount($quote->getRewardCurrencyAmount() + $pointsCurrencyAmountUsed);
        $quote->setBaseRewardCurrencyAmount($quote->getBaseRewardCurrencyAmount() + $basePointsCurrencyAmountUsed);

        $total->setRewardPointsBalance($pointsBalanceUsed);
        $total->setRewardCurrencyAmount($pointsCurrencyAmountUsed);
        $total->setBaseRewardCurrencyAmount($basePointsCurrencyAmountUsed);
    }
    return $this;
}
{% endhighlight %}

This is where the magic happens. As you can see on line 24, the same `reward_instance` field is referenced here. Since this was set in the `Magento\Reward\Model\RewardManagement::set` function, the next if block is ignored. If it was not set, it would load the reward points record using the quote's customer ID.

Starting on line 38, it checks to see if the amount of reward points the customer has in their balance is more than the grand total of the order. If it is, line 39 sets `$pointsBalanceUsed` to the points equivalent of the order grand total. If the reward points balance is less than the grand total of the order, `$pointsBalanceUsed` is set to the points equivalent of the remainder of reward points currency. That last sentence sounds confusing, but you can see on line 37 that `$baseRewardCurrencyAmountLeft` is set to the customer's reward points balance in currency form minus the amount of reward point currency already used in the quote, so either way the if statement on line 39 goes, `$pointsBalanceUsed` will always be set the maximum amount possible to cover the order grand total. Finally, `$total->setGrandTotal()` and `$total->setBaseGrandTotal()` are both called to subtract the amount of reward points redeemed from the original grand total.


This blog post is the first of a series of blog posts I hope to make in the future about Magento 2. Thanks for reading, and I hope this helps someone who wants to learn more about how Magento 2 works under the hood.