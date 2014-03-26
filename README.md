<h1> Comment Widget</h1>

<p>CommentWidget is an easy-to-use, platform-independent, cross-browser-compatible, open-source, ajax, threaded comment widget.</p>

<h1> Comment Widget - Easy Ajax Comments </h1>
  <ol>
    <li><strong>Platform Independent</strong> - It doesn't matter what back-end you
        use - CommentWidget just need a few URLs that return information in JSON.
    <li><strong>Cross-Browser Compatible</strong> - Tested in IE6/7,
  FF2/3, and Safari 3.
    <li><strong>Ajax</strong> - Let people comment without reloading
  the page.
    <li><strong>Threaded</strong> - Comments can be discussions.
    <li><strong>Open Source</strong> - Free, open, and extensible, forever.
  </ol>
  
  <h1> Comment Widget - Getting Started </h1>
  CommentWidget is a small Javascript library that displays comment
  threads. CommentWidget is server agnostic. CommentWidget
  communicates with a server to get and post comments via JSON. Setting
  up a CommentWidget on the front end is very simple. You
  simply include the CommentWidget script in the page, and
  instantiate the widget with a single line of Javascript.

  <p>The caveat is you need to set up your particular server to store and use comments.
  Below, you can see the code for a Django/Python project
  configured to use CommentWidget. However, CommentWidget
  really doesn't care what kind of database, web framework, or other
  back-end macinery you use. It just needs a place to store comments
  and three URLs where it can communicate with your server and get
  JSON responses.</p>

  <h2>Set Up Your Server (Django Example Code)</h2>
 <h3>Database Tables</h3>
  In our example, we have a model to store comments, and a model to
  store webpages, which have threads of comments.  We just have comments
  about webpages in this example, but you can see an <a href=/web/20130522115231/http://www.trailbehind.com/comment_widget/multiple_objects>example
  with comments about multiple things</a>.
 <pre>class Comment(models.Model):
  comment = models.TextField(blank=True)    
  author = models.ForeignKey("User", null=True)
  time_created = models.DateTimeField(default=datetime.now)
  parent = models.ForeignKey('self', related_name="parent", null=True, blank=True)
  #generic relation fields
  object_id = models.PositiveIntegerField(db_index=True)
  content_type = models.ForeignKey(ContentType)
  content_object = generic.GenericForeignKey()
  

class Webpage(models.Model): 
  url = models.URLField(null=True)
  comments = generic.GenericRelation(Comment, null=True, blank=True)
  </pre>
<h3>URLs</h3>
  <p>
  Along with the models, we specify URLs for 1) posting a comment about a webpage, 2) getting comments about a webpage, and 3) posting a reply to a comment. 
  </p>
<pre>
from django.conf.urls.defaults import *
from thetrailbehind.apps.comments import views
urlpatterns = patterns('',
        (r'^get_comments/webpage/(?P<id>[^/]+)$', views.get_webpage_comments),
        (r'^post_comment/webpage/(?P<id>[^/]+)$', views.post_webpage_comment),
        (r'^post_reply/(?P<comment_id>[^/]+)$', views.post_reply),
    )
</pre>

<h3>Server Functions</h3>

In each of the URLs, the server always returns a JSON list of comments that looks like:
<pre>{"comments": [{"comment": string, "id":int,"parent":int, "author": string, "time_created": string}, ...]}</pre>

<p>Here are the views that underly the URLs.</p>

<pre>
from django.util import simplejson
from django.http import HttpResponse
from thetrailbehind.apps.site.models import *

def post_webpage_comment(request, id):
  '''post a comment about a webpage'''
  webpage = Webpage.objects.get(id=id)
  myComment = request.POST.get("comment")
  return save_comment_to_db(request, myComment, None, webpage)

def post_reply(request, comment_id):
  '''This saves a Comment when the user enters a reply to a comment, similar to PostComment()'''
  reply_div = "replyDiv" + str(comment_id)
  myComment = request.POST.get(reply_div)
  reply_comment = Comment.objects.get(id=comment_id)
  object = reply_comment.content_object
  return save_comment_to_db(request, myComment, reply_comment, object)

def save_comment_to_db(request, myComment, parent, comment_object):
  '''saves a comment to the database and returns a list of comments about comment_object'''
  newComment = Comment()  
  if parent:
    newComment.parent = parent
  newComment.content_object = comment_object
  newComment.comment = myComment
  if str(request.user) != 'AnonymousUser':
    newComment.author = UserProfile.objects.get(user=request.user)
  newComment.save()    
  return get_comments(request, comment_object)

def get_webpage_comments(request, id):
  '''returns a JSON list of comments about a webpage'''
  webpage = Webpage.objects.get(id=id)
  return get_comments(request, webpage)

def get_comments(request, object):
  '''returns a JSON dictionary of comments about the object'''
  comments = object.comments.all().order_by('-id')        
  comments = make_json_comments(comments)
  response_dict = {'success':True, 'comments':comments}
  return HttpResponse(simplejson.dumps(response_dict), mimetype='application/javascript')

def make_json_comments(comments):
  '''turn a queryset of comments into a JSON list'''
  json_comments = []
  for comment in comments:
    new_comment = {}
    new_comment['id'] = comment.id
    new_comment['comment'] = comment.comment
    try:
      new_comment['parent'] = comment.parent.id
    except:
      new_comment['parent'] = None
      new_comment['author'] = str(comment.author)
    json_comments.append(new_comment);  
  return json_comments       
</pre>

  <h2>How the Javascript Works</h2>
  <h3>Include the Script</h2>
  You probably want to include comments.js and comments.css at the end of your HTML
  template, so they will load after the rest of your page. Just do the includes before you close the HTML tag. Or you can load the files from our servers, and get updates automatically when we release them:

  <pre>
&lt;link rel="stylesheet" type="text/css" href="path/to/comments.css">
&lt;script type="text/javascript" src="/path/to/comments.js"></script> </pre>
<pre>&lt;link rel="stylesheet" type="text/css" href="http://www.trailbehind.com/site_media/css/comments.css">
&lt;script type="text/javascript" src="http://www.trailbehind.com/site_media/javascript/comments.js"></script></pre>
  <h3>Create and Initialize the Widget</h3>
  Creating a CommentWidget only requires
  one parameter - the id of the div on the page where the comment
  widget should be rendered. Optionally, you can also pass parameters to
  specify different base URLs than the defaults.

  <pre>var commentWidget = new CommentWidget('comments')</pre>

  Then, call commentWidget.select to specify target for the widget. The select function sends an ajax request to /comments/get_comments/webpage/1, and gets back a list of comments in JSON format.
  <pre>commentWidget.select('webpage', 1);   //sends an ajax request to /comments/get_comments/webpage/1
  </pre>
  <h3>Posting Comments and Replies</h3>
  <p>CommentWidget also communicates with the server when there is a comment or reply posted. Assuming that a 'webpage' with id '1' was selected as above, when a comment is posted, the widget sends a request to:

  <pre>/comments/post_comment/webpage/1</pre>

  <p>The server-side of this URL should first save the comment and author from the
  request(see the above Django views), and then it should return a JSON list of comments.</p>

  <p>Similarly, when a reply is posted, the widget sends a request to:</p>
  <pre>/comments/post_reply/{some_id}</pre>

  The server-side of this URL first saves the reply (see the Django views above). Then it return a JSON list of comments as usual.
<pre>{"comments": [{"comment": string, "id":int,"parent":int, "author": string, "time_created": string}, ...]}</pre>

<h1> Comment Widget - Additional Docs </h1>
<h1> Comment Widgets with Multiple Objects </h1>
 Many sites have multiple things to be commented on. For example,
 you might want there to be comments attached to blogposts and
 webpages. That's why the select function of CommentWidget takes
 a string.
 <pre>commentWidget.select('webpage', 1);   //sends an ajax request to /comments/get_comments/webpage/1</pre>
 <p>Here's how you might set up the back-end of a Django site for
 comments on webpages AND blogposts: 
 <h3>Database Tables</h3>
 <pre>class Comment(models.Model):
  comment = models.TextField(blank=True)    
  author = models.ForeignKey("User", null=True)
  time_created = models.DateTimeField(default=datetime.now)
  parent = models.ForeignKey('self', related_name="parent", null=True, blank=True)
  #generic relation fields
  object_id = models.PositiveIntegerField(db_index=True)
  content_type = models.ForeignKey(ContentType)
  content_object = generic.GenericForeignKey()
  
class Webpage(models.Model): 
  url = models.URLField(null=True)
  comments = generic.GenericRelation(Comment, null=True, blank=True)

class Blogpost(models.Model): 
  text = models.TextField(blank=True)
  comments = generic.GenericRelation(Comment, null=True, blank=True)
  </pre>
<h3>URLs</h3>
<pre>
from django.conf.urls.defaults import *
from thetrailbehind.apps.comments import views
urlpatterns = patterns('',
        (r'^get_comments/webpage/(?P<id>[^/]+)$', views.get_webpage_comments),
        (r'^get_comments/blogpost/(?P<id>[^/]+)$', views.get_blogpost_comments),
        (r'^post_comment/webpage/(?P<id>[^/]+)$', views.post_webpage_comment),
        (r'^post_comment/blogpost/(?P<id>[^/]+)$', views.post_blogpost_comment),
        (r'^post_reply/(?P<comment_id>[^/]+)$', views.post_reply),
    )
</pre>

<h3>Server Functions</h3>

<pre>
from django.util import simplejson
from django.http import HttpResponse
from thetrailbehind.apps.site.models import *

def post_webpage_comment(request, id):
  '''post a comment about a webpage'''
  webpage = Webpage.objects.get(id=id)
  myComment = request.POST.get("comment")
  return save_comment_to_db(request, myComment, None, webpage)

def post_blogpost_comment(request, id):
  '''post a comment about a blogpost'''
  blogpost = Blogpost.objects.get(id=id)
  myComment = request.POST.get("comment")
  return save_comment_to_db(request, myComment, None, blogpost)

def post_reply(request, comment_id):
  '''This saves a Comment when the user enters a reply to a comment, similar to PostComment()'''
  reply_div = "replyDiv" + str(comment_id)
  myComment = request.POST.get(reply_div)
  reply_comment = Comment.objects.get(id=comment_id)
  object = reply_comment.content_object
  return save_comment_to_db(request, myComment, reply_comment, object)

def save_comment_to_db(request, myComment, parent, comment_object):
  '''saves a comment to the database and returns a list of comments about comment_object'''
  newComment = Comment()  
  if parent:
    newComment.parent = parent
  newComment.content_object = comment_object
  newComment.comment = myComment
  if str(request.user) != 'AnonymousUser':
    newComment.author = UserProfile.objects.get(user=request.user)
  newComment.save()    
  return get_comments(request, comment_object)

def get_webpage_comments(request, id):
  '''returns a JSON list of comments about a webpage'''
  webpage = Webpage.objects.get(id=id)
  return get_comments(request, webpage)

def get_blogpost_comments(request, id):
  '''returns a JSON list of comments about a webpage'''
  blogpost = Blogpost.objects.get(id=id)
  return get_comments(request, blogpost)

def get_comments(request, object):
  '''returns a JSON dictionary of comments about the object'''
  comments = object.comments.all().order_by('-id')        
  comments = make_json_comments(comments)
  response_dict = {'success':True, 'comments':comments}
  return HttpResponse(simplejson.dumps(response_dict), mimetype='application/javascript')

def make_json_comments(comments):
  '''turn a queryset of comments into a JSON list'''
  json_comments = []
  for comment in comments:
    new_comment = {}
    new_comment['id'] = comment.id
    new_comment['comment'] = comment.comment
    try:
      new_comment['parent'] = comment.parent.id
    except:
      new_comment['parent'] = None
      new_comment['author'] = str(comment.author)
    json_comments.append(new_comment);  
  return json_comments       
</pre>

<h1> Changing Default Parameters </h1>
 <h2>Request URLs</h2>
 By default, CommentWidget sends requests to the server at the
 following URLs:
 <pre>
   /comments/get_comments/{some_string}/{some_int}
   /comments/post_comment/{some_string}/{some_int}
   /comments/post_reply
 </pre>

 You can change the base URLs of these requests when you instantiate
 the CommentWidget:

 <pre>var commentWidget = CommentWidget('foo', '/my/get/path',
 'my/post/path', 'my/reply/path');
 </pre>

 Then, CommentWidget will send requests as follows:

 <pre>
   /my/get/path/{some_string}/{some_int}
   /my/post/path/{some_string}/{some_int}
   /my/reply/path/post_reply
 </pre>

 <h2>Form Text</h2>
 By default, CommentWidget use the following text for the default text
 in the text form:
 <pre>Your comment goes here.</pre>

 You can change this when you instantiate the CommentWidget:

 <pre>var commentWidget = CommentWidget('foo', false, false, false, 'Your custom text.');</pre>

<h1> Saving the Author of Comments </h1>
 By default, comments are posted anonymously and show up as 'by
 AnonymousUser.' However, if a user is logged in, you'll probably want
 to store and display that user as the author of the comment. Here's how we do it
 in the example code:

 <pre>
def save_comment_to_db(request, myComment, parent, comment_object):
  '''saves a comment to the database and returns a list of comments about comment_object'''
  newComment = Comment()  
  if parent:
    newComment.parent = parent
  newComment.content_object = comment_object
  newComment.comment = myComment
  <strong>if str(request.user) != 'AnonymousUser':
    newComment.author = UserProfile.objects.get(user=request.user)</strong>
  newComment.save()    
  return get_comments(request, comment_object)
</pre>

 <p>TO DO: Add an option to CommentWidget to use a Name form field and a
 captcha, instead of requiring website designers to integrate with
 their login system.</p>


<h1> Skinning the CommentWidget</h1>
 The CommentWidget comes with a very basic CSS. You can skin the
 widget however you want by overriding the CSS in your HTML template:

<pre>
.clickedTextBox {
  height: 60px;
}


.commentText {
  width: 96%; 
  margin-top: 10px; 
  margin-bottom: 10px;  
}


.replyText {
  margin-bottom: 5px; 
  width: 96%;
}


a.jLink {
  color: #27408B;
  cursor: pointer;
  text-decoration: underline;
}
</pre>



  <h2>Roadmap/Bug List</h2>
  <ol>
    <li>Make a sample Django project to demo CommentWidget.
    <li>Add voting/moderation.
    <li>Add integrated login widget.
    <li>Make comments sortable by date, votes, etc.
    <li>More detailed API documentation.
    <li>Make default reply text a parameter.
    <li>Option to use name form and captcha instead of requiring login integration.
    <li>Add a README file to the zip.
    <li>Do better minimization of comments-min.js.
    <li>Release a YUI-integrated version.
  </ol>
<h2>Open Source License</h2>
BSD-licensed.
