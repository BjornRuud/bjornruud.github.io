---
layout: post
title: Authorize MoinMoin Users With WordPress DB
tags:
- moinmoin
- mysqldb
- phpass
- python
- wiki
- wordpress
status: publish
type: post
published: true
excerpt: In order to authorize MoinMoin wiki users with a WordPress user database, MoinMoin needs a custom authorization plugin. Two packages are required for it to work: MySQLdb for MySQL support, and the PHPass Python module for bcrypt password hash support so we can compare stored and given password.
---
(Note: This article was originally posted on my old [Wikidot](http://www.wikidot.com/) site on 2009-10-17)

In order to authorize [MoinMoin](http://moinmo.in/) wiki users with a [WordPress](http://wordpress.org/) user database, MoinMoin needs a custom authorization plugin. Two packages are required for it to work: [MySQLdb](http://sourceforge.net/projects/mysql-python) for MySQL support, and the [PHPass](http://www.openwall.com/phpass/) Python module for bcrypt password hash support so we can compare stored and given password.

The following is the necessary code from `wikiconfig.py`. You can put this anywhere you want as long as Python can find it, but sticking it in `wikiconfig.py` is an easy solution.

```python
class WordpressAuth(BaseAuth):
    """Authenticate users with a Wordpress database.

    This class has the following module requirements:
    - MySQLdb (http://sourceforge.net/projects/mysql-python)
    - phpass (http://www.openwall.com/phpass/)
    """

    login_inputs = ['username', 'password']
    logout_possible = True
    name = 'wordpress'

    def __init__(self, autocreate=False,
                 db_host='localhost', db_name='',
                 db_user='', db_passwd=''):
        """Initialize class.

        autocreate -- When set to False Moin user accounts are not updated
                      or created.
        db_host -- Hostname where Wordpress DB resides.
        db_name -- Name of Wordpress DB.
        db_user -- Name of DB account with read rights to the wp_user table.
        db_passwd -- Password of DB account.
        """

        BaseAuth.__init__(self)
        self.autocreate = autocreate
        self.db_host = db_host
        self.db_name = db_name
        self.db_user = db_user
        self.db_passwd = db_passwd

    def login(self, request, user_obj, **kw):
        """Handle Moin login request, using Wordpress DB for authentication."""

        username = kw.get('username')
        password = kw.get('password')
        _ = request.getText

        # Continue if user is already valid
        if user_obj and user_obj.valid:
            return ContinueLogin(user_obj)

        if not (username and password):
            return ContinueLogin(user_obj, _("Please enter both user name and password."))

        # Authenticate user and password against Wordpress DB
        row = self.verifyUser(username, password)
        if row:
            user_login, user_pass, user_nicename, user_email = row
            u = user.User(request,
                          auth_username=username,
                          auth_method=self.name,
                          auth_attribs=('name', 'password', 'email', 'aliasname'))
            u.name = user_login
            u.aliasname = user_nicename
            u.email = user_email
            if self.autocreate:
                u.create_or_update(True)
            return ContinueLogin(u)

        return ContinueLogin(user_obj, _("Invalid username or password."))

    def verifyUser(self, username='', password=''):
        """Verify a username and password with a Wordpress database.

        This implementation assumes passwords are hashed using bcrypt, and are
        PHPass compatible.

        username -- Name of Wordpress user.
        password -- Password of Wordpress user.
        """

        import MySQLdb
        db = MySQLdb.connect(host=self.db_host,
                             db=self.db_name,
                             user=self.db_user,
                             passwd=self.db_passwd)
        c = db.cursor()
        q = "SELECT user_login, user_pass, user_nicename, user_email
             FROM wp_users WHERE user_login='%s'" % (username)
        result = c.execute(q)
        if result == 1:
            row = c.fetchone()
            import phpass  # A bit ugly, but the module is only needed here
            if row and (row[1] == phpass.crypt_private(password, row[1])):
                result = row
            else:
                result = None
        else:
            result = None

        c.close()
        return result

class Config(DefaultConfig):
#~ class FarmConfig(DefaultConfig):

    auth = [WordpressAuth(autocreate=True,
                          db_host='localhost', db_name='wpdb',
                          db_user='wpuser', db_passwd='wppass')]

    # don't let the user change these, but show them
    user_form_disable = ['name', 'aliasname', 'email']
    # remove completely
    user_form_remove = ['password', 'password2', 'css_url',
                        'create', 'account_sendmail','jid']
    # Is this necessary?
    user_autocreate = True
```

[Download example wikiconfig.py](/assets/2011/02/wikiconfig-wordpress-auth.zip)

The authorization plugins included with MoinMoin are good examples of what you can do, so study them if you want to customize something or write your own.

If we wanted a single sign-on solution we would probably have to use the WordPress cookie in the authentication process. For security we should then store the WordPress session ID in the database and authenticate by comparing that with the session ID in the cookie.
