FAQ
====

Internationization
-----------------

**Will any other languages besides English be officially supported?**

Due to high development workloads, only support English is supported at the moment and unfortunately we don't have the capacity to add or manage other languages.

That being said, we're working on an improved translation system that will allow easy integration and maintenance of community maintained translations, however, this is still very much a work in progress and an ETA has not yet been set.

**Can I add other translations myself?**

Soon documentation on how to add additional languages will be available. For the time being, it's not possible yet to add additional languages yourself.


Integrations
------------

**Can Costasiella be integrated in a Squarespace or Wix website?**

The short answer is no. As these website builders operate a closed platform, the options for integration are limited until there's an official plugin (which is very unlikely unfortunately, as only very big companies get an official plugin). There is an option for third parties to inject code, but the supported technologies aren't sufficient for a satisfying integration. In case you'd like to integrate Costasiella on your website, it's probably best to move to a more open system like WordPress, Drupal or CMS made simple to name a few.

**Are other online payment providers supported?**

Mollie is the only supported payment provider at the moment. This means that you won't be able to use online payments if you're based outside of the EU. More integrations are planned. Please let us know which ones you'd like to see, they'll be added in order of populate demand. In case you'd like to sponsor the integration of a payment provider by financial support of by donating code, please get in touch using the contact page!

Self hosted - DIY
------------------

**Are 2 (v)CPU cores really necessary when hosting it yourself?**

Well, the answer is yes and no. Technically you can run Costasiella on a single CPU core. Though you will find that Costasiella will run quite slow at times and pages might take more then a few seconds to load.
This is caused by the fact that you can only run one app worker process comfortable per core. With one core and one process serving requests, it means that when the process is busy, all other requests will have to wait.
Imagine the case where someone using a mobile phone using a poor connection tries to load a page with a few images like the list of events. The phone will make one request for the page itself and one for each image. Until all requests are handled, other users will have to wait. However this is in case there is only 1 CPU core and 1 app worker process. With a 2nd CPU core and worker process additional requests can be handled without everyone having to wait on the user with the poor connection. Another benefit is much quicker loading times in general. In many cases a few requests need to be made before a page can be shown. The page itself, style sheets, scripts and images are all separate requests. The more worker processes, the more requests can be handled in concurrently.
Although this isn't true in all cases, for the sake of simplicity you could say that with a single CPU core and app worker process requests are handled one by one, in a serial way and with multiple CPU cores and app worker processes requests can be handled in parallel, greatly reducing loading times of even single pages.

