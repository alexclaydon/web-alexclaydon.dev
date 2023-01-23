I recently went through the process of deploying a Django project to Heroku. This is my first Heroku deployment. I had also considered a number of alternatives including [dokku on Digital Ocean](https://marketplace.digitalocean.com/apps/dokku), [piku](https://github.com/piku) (also on Digital Ocean) and deploying to AWS Lambda using [Zappa](https://github.com/Miserlou/Zappa). While all of these are interesting in their own way (not least of which, in the case of Dokku / Piku is the absence of vendor lock-in) - and I intend to try each of them in due course - I wanted to take a look at Heroku first, giving that it comes up regularly on hackernews.

But let's backtrack for a second: why not simply deploy the Django project to bare metal - perhaps one of the many Raspbery Pi 4s I have lying around - or to a generic Digital Ocean droplet? Because while there are a range of excellent reasons to do that, I am trying to be very specific about targeting where I want to sit in terms of expertise in the stack. I am tring to make sensible decisions about which parts of the stack I can hand off to a managed service, and I think Heroku and Dokku, in particular, are solid, 'boring', choices - I'll save my [innovation tokens](https://mcfunley.com/choose-boring-technology) for something else.

## The deployment

I followed [this](https://blog.heroku.com/from-project-to-productionized-python) article, which I find to be generally very good, although there were a couple of hitches for me and my particular stack that I thought would be helpful to share for anyone trying to do the same thing.

Here's the stack:
- Django 3.0.8
- Github-hosted project repo
- A Django project created with Pycharm's Django project template
- A domain hosted on porkbun.com

And here are my notes:
- One minor addition that you have to make for any app that's been created using the Pycharm Django project template - and that's not diectly covered in the article - is to add `STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')` to your base settings file (assuming that you're following the convention recommended by the Heroku article, that would be located at `mysite.settings.base.py`) and to make a `staticfiles` directory at your project root (i.e., the directory that contains both your project and your app subdirectories). These changes are required to ensure that the `whitespace` middleware works properly at build time. You may also need to put a `.gitkeep` in that directory to ensure that git picks it up even when it's empty.
- You will need to have run `heroku config:set ALLOWED_HOSTS=".herokuapp.com"` at the command line in order for the call to `env.list('ALLOWED_HOSTS')` in your `mysite.settings.heroku.py` settings file to correctly permit web access to the deployed app. See [this](https://stackoverflow.com/questions/23252733/i-get-an-error-400-bad-request-on-custom-heroku-domain-but-works-fine-on-foo-h) Stack Overflow question. Don't forget to also run `heroku config:set ALLOWED_HOSTS="<your-domain.com>"` for any external domains you point at your project.
- You can enable push-to-deploy for Github-hosted repos by going to the "Deploy" tab in the Heroku admin page and simply connecting it to the repo, then separately enabling automatic deployment (for pushes to your production branch only, obvs).
- Before being able to access the admin panel on the live site, you'll need to have added a new superuser; invoke `heroku run python manage.py createsuperuser` from the command line and enter your details.
- To add a Porkbun-hosted domain to a Heroku app, see [this](https://kb.porkbun.com/article/90-how-to-connect-herokuapp-with-porkbun) article, but note that in addition to what's set out in there, you also have to delete the existing `ALIAS` record for the domain that points at `pixie.porkbun.com` (i.e., you need to not only add ALIAS and CNAME records, but also delete one existing ALIAS record (assuming your domain is still configured with the Porkbun default settings).
- After adding any custom domains, don't forget to provision a SSL certificate (using the automated ("ACM") option on the Heroku admin panel).
- Once the SSL certificate is provisioned, you'll want to enable routing of all http requests to https. See [this](https://help.heroku.com/J2R1S4T8/can-heroku-force-an-application-to-use-ssl-tls), but in short you need to add the following two lines to your `base/py` settings file:
```
  SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
  SECURE_SSL_REDIRECT = True
```
- You should consider at what stage of your deployment pipeline you will have your tests run. In addition to local tests, I'm inclined to require testing using Github actions (as a pre-push hook) as that approach generalises to all of my repos, not just those destined for deployment on Heroku.