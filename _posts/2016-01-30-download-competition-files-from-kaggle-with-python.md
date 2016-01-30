---
layout: post
title:  "Download competition data from Kaggle with BeautifulSoup and Mechanize"
tags: [python, kaggle, beautifulsoup, mechanize]
date:   2016-01-30 19:00:00
image:
  feature: cover.jpg
---

There might be several reasons why you need to get files from Kaggle via script. In my case I was playing with Theano and Lasagne and wanted to download data directly to AWS GPU instance.

Kaggle doesn't provide an API so we have to emulate real browser and user. Fortunately there is mechanize - stateful programmatic web browser for Python. It's not supported since 2012 but works almost fine. I encountered only one problem so far - it reuses previously created request for form submission and if you are redirected to a current page from some other it will send incorrect referrer header. Facebook will consider this as a phishing and won't allow you to login. My solution was to assign a request a None value in submit method, which looks a bit dirty but is the fastest way to fix it.

For the simplicity lets extend `mechanize.Browser` class to be able to reuse it's methods and object class to invoke parent's methods explicitly.

{% highlight python %}
class KaggLoader(mechanize.Browser, object):
    BASE_DIR = 'kaggloader'
    COOKIE_PATH = os.path.join(BASE_DIR, '.cookies')
    BASE_URL = 'https://www.kaggle.com'

    def __init__(self, login=login.facebook):
        super(KaggLoader, self).__init__()

        self.login = login

        self.set_handle_equiv(True)
        self.set_handle_robots(False)

        if not os.path.exists(self.BASE_DIR):
            os.makedirs(self.BASE_DIR)

        if not os.path.exists(self.COOKIE_PATH):
            with open(self.COOKIE_PATH, 'w') as f:
                f.write('#LWP-Cookies-2.0')

        self.cj = mechanize.LWPCookieJar()
        self.cj.load(self.COOKIE_PATH, ignore_discard=False, ignore_expires=False)
        opener = mechanize.build_opener(mechanize.HTTPCookieProcessor(self.cj))
        mechanize.install_opener(opener)

        self.set_cookiejar(self.cj)
{% endhighlight %}

Constructor creates a directory to download files and a file to story cookies if they don't exist yet. It accepts only 1 argument - the login strategy, which is simply a function implementing one of the authentication mechanisms available on Kaggle. Here is the example of the Facebook login:

{% highlight python %}
def facebook(browser):
    def is_fb_login_form(form):
        return form.attrs.get('method') == 'post' and form.action.find('/login.php') != -1

    browser.addheaders = [('User-Agent',
                        'Mozilla/5.0 (iPhone; CPU iPhone OS 8_1 like Mac OS X) AppleWebKit/600.1.4 (KHTML, like Gecko) Version/8.0 Mobile/12B410 Safari/600.1.4')]

    browser.follow_link(url_regex=r"facebook", nr=1)
    browser.select_form(predicate=is_fb_login_form)
    browser.form['email'] = raw_input("Email: ")
    browser.form['pass'] = getpass.getpass()
    browser.submit()
{% endhighlight %}

I set User-Agent header to the same value as Safari on iPhone because mobile version of Facebook can work without Javascript, but desktop version can't.

`KaggLoader` overwrites `open` method of `mechanize.Browser` and extends it with single line of code to store cookies like every regular browser and avoid entering login and password every time we run the script.

{% highlight python %}
def open(self, url):
    response = super(KaggLoader, self).open(url)
    self.cj.save(self.COOKIE_PATH, ignore_discard=False, ignore_expires=False)
    return response
{% endhighlight %}

First of all we have to get the list of data files available for the competition. This page is publicly available so we can achieve this by merely parsing it.

{% highlight python %}
def get_files(self, competition):
    self.open(self.BASE_URL + '/c/' + competition + '/data')

    soup = BeautifulSoup(self.response().read(), "html.parser")

    for anchor in soup.find('table', id='data-files').find_all('a'):
        yield anchor['href'].replace('/c/' + competition + '/download/', '')
{% endhighlight %}

Then we iterate over the list of files and download them one by one.

{% highlight python %}
def download_all(self, competition):
    for file in self.get_files(competition):
        self.download(competition, file)
{% endhighlight %}

Kaggle won't allow us to download anything before we sign in and accept the rules, therefore the algorithm is the following:

1. Attempt to download the file.
2. If Kaggle displays login page then login and go to step 1, otherwise go to step 3.
3. If rules page is shown then accept rules and go to step 1, else go to step 4.
4. Read a response and store it in a file.

{% highlight python %}
def download(self, competition, file_name):
    response = self.open(self.get_url(competition, file_name))

    if self.not_logged_in():
        self.login(self)
        self.download(competition, file_name)
    elif self.rules_not_accepted():
        self.accept_rules()
        self.download(competition, file_name)
    else:
        dir = os.path.join(self.BASE_DIR, competition)
        if not os.path.exists(dir):
            os.makedirs(dir)

        with open(os.path.join(dir, file_name), 'wb') as f:
            f.write(response.read())
{% endhighlight %}

Usage example:

{% highlight python %}
from kaggloader import KaggLoader

competition = 'titanic'

loader = KaggLoader()
loader.download_all(competition)
{% endhighlight %}

Here are some improvements I'm planning:

1. More login strategies: Kaggle, MS, Yahoo.
2. Automatic archive extraction.
3. IPhython notebook plugin.

However if don't want to wait or have other ideas fork it on [GitHub](https://github.com/Sundrique/kaggloader). 