---
layout: page
title: Contact
---
# Contact Me

Twitter: [@BjornRuud](https://twitter.com/BjornRuud)

<form class="pure-form pure-form-stacked" method="POST" action="http://www.domeneshop.no/cgi-bin/mailto.cgi" accept-charset="UTF-8">
  <fieldset>
    <input type="hidden" name="_to" value="contact@bjornruud.net">
    <input type="hidden" name="_resulturl" value="{{ site.url }}">
    <label>Email</label>
    <input type="text" name="_from" placeholder="Your email" required style="width: 300px">
    <label>Subject</label>
    <input type="text" name="_subject" placeholder="Subject" required style="width: 300px">
    <label>Message</label>
    <textarea name="message" placeholder="Message" required style="width: 300px; height: 200px"></textarea>
    <button type="submit" class="pure-button pure-button-primary">Send</button>
  </fieldset>
</form>
