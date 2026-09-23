+++
title = "How to Add Google AdSense to Your Hugo Website"
date = "2024-04-20"
image = "/images/how_to_add_google_adsense_to_your_hugo_website.webp"
tags = [
"hugo",
"google",
"adsense",
"development",
]
categories = [
"Development",
]
+++

Adding Google AdSense to your Hugo website can help you monetize your content and earn revenue through ads. Follow these simple steps to integrate AdSense seamlessly into your site.

## Step 1: Sign up for Google AdSense

First, if you haven't already, sign up for a Google AdSense account at [www.google.com/adsense](www.google.com/adsense). Once your account is approved, you'll receive a code snippet to add AdSense to your website.

## Step 2: Generate AdSense Code

Log in to your AdSense account and navigate to the "Ads" tab. Click on "By ad unit" and then select "Display ads" or any other ad type you prefer. Customize the ad size, style, and other settings to match your website's design.

After customizing, click on "Save and get code." Copy the generated AdSense code snippet.

## Step 3: Edit Your Hugo Website

Open your Hugo website's source code in your preferred code editor. Navigate to the layout file where you want to display AdSense ads, such as `layouts/partials/footer.html` for a footer ad.

```html
<!-- Start of Google AdSense -->
<script
  async
  src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-XXXXXXXXXXXXXXX"
  crossorigin="anonymous"
></script>
<!-- End of Google AdSense -->
<!-- Google Tag Manager -->
<script>
  (function (w, d, s, l, i) {
    w[l] = w[l] || [];
    w[l].push({ "gtm.start": new Date().getTime(), event: "gtm.js" });
    var f = d.getElementsByTagName(s)[0],
      j = d.createElement(s),
      dl = l != "dataLayer" ? "&l=" + l : "";
    j.async = true;
    j.src = "https://www.googletagmanager.com/gtm.js?id=" + i + dl;
    f.parentNode.insertBefore(j, f);
  })(window, document, "script", "dataLayer", "GTM-XXXXXXXXXXXXXXX");
</script>
<!-- End Google Tag Manager -->
```

Paste the copied AdSense code snippet into the appropriate location within the HTML code. Save the changes.

## Step 4: Test Your Changes

After saving the changes, run `hugo server` to test your website locally. Navigate to the page where you added the AdSense code to ensure the ads display correctly (you can use the browser `inspect` for that).

## Step 5: Deploy Your Hugo Website

Once you've verified that the AdSense ads are displaying correctly, deploy your Hugo website to make the changes live. Use your preferred hosting provider or platform to publish your website.

## Step 6: Monitor AdSense Performance

Log in to your Google AdSense account to monitor the performance of your ads, including impressions, clicks, and earnings. Optimize your ad placement and settings based on performance data.

That's it! You've successfully added Google AdSense to your Hugo website. Start earning revenue from your content through AdSense ads.

![Diargram](/images/how-to-add-google-adsense-to-your-hugo-website-diargram.png)
