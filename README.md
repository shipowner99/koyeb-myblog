# 블로그 사용자 추가하기

### 요구 사항 1 - 신규 사용자 등록
1. 어제 배운 내용을 활용하여 사용자가 /register 경로로 이동해서 블로그 웹 사이트에 등록할 수 있도록 합니다. forms.py에 RegisterForm이라는 WTForm을 만들고 플라스크 부트스트랩을 사용하여 wtf quick_form을 렌더링해야 합니다.

사용자가 입력한 데이터를 사용하여 User 테이블의 blog.db에 새 항목이 만들어져야 합니다.
```python
#forms.py
class RegisterForm(FlaskForm):
    email = StringField("Email", validators=[DataRequired()])
    password = PasswordField("Password", validators=[DataRequired()])
    name = StringField("Name", validators=[DataRequired()])
    submit = SubmitField("Sign Me Up!")
```
```python
#register.html
{{ wtf.quick_form(form, novalidate=True, button_map={"submit": "primary"}) }}
```
```python
#main.py
#Import RegisterForm from forms.py
from forms import CreatePostForm,  RegisterForm

#Create the User Table
class User(UserMixin, db.Model):
    __tablename__ = "users"
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(100), unique=True)
    password = db.Column(db.String(100))
    name = db.Column(db.String(100))
    
# Create all the tables in the database
with app.app_context():
    db.create_all()

#Register new users into the User database
@app.route('/register', methods=["GET", "POST"])
def register():
    form = RegisterForm()
    if form.validate_on_submit():

        hash_and_salted_password = generate_password_hash(
            form.password.data,
            method='pbkdf2:sha256',
            salt_length=8
        )
        new_user = User(
            email=form.email.data,
            name=form.name.data,
            password=hash_and_salted_password,
        )
        db.session.add(new_user)
        db.session.commit()
        
        return redirect(url_for("get_all_posts"))

    return render_template("register.html", form=form)
```

### 요구 사항 2 - 등록된 사용자 로그인
1. 성공적으로 등록이 완료된(데이터베이스의 사용자 테이블에 추가된) 사용자는 login 경로로 이동하여 자신의 크리덴셜(자격 증명 정보)을 사용하여 로그인할 수 있어야 합니다. 그러려면 Flask-Login 문서와 어제 배운 내용을 활용해야 합니다.
```python
#forms.py
class LoginForm(FlaskForm):
    email = StringField("Email", validators=[DataRequired()])
    password = PasswordField("Password", validators=[DataRequired()])
    submit = SubmitField("SIGN ME UP!")
```
```python
from flask_login import UserMixin, LoginManager

login_manager = LoginManager()
login_manager.init_app(app)

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))


```

2. /register 경로에 코드 한 줄을 추가해서 사용자 등록이 성공적으로 완료된 경우 해당 사용자를 다시 홈페이지로 보내 Flask-Login으로 로그인하도록 만들어보세요.
```python
login_user(new_user)
return redirect(url_for('get_all_posts'))
```

3. /register 경로에서 사용자가 데이터베이스에 이미 존재하는 이메일로 등록을 시도하는 경우, /login 경로로 리디렉션하고 해당 이메일로 로그인하라는 플래시 메시지가 표시되어야 합니다.

```python

#main.py
if User.query.filter_by(email=form.email.data).first():
    flash("You've already signed up with that email, log in instead!")
    return redirect(url_for('login'))
else: ## MORE CODE BELOW

#login.html
<div class="container">
    <div class="row">
        <div class="col-lg-8 col-md-10 mx-auto content">
            {% with messages = get_flashed_messages() %}
            {% if messages %}
            {% for message in messages %}
            <p>{{ message }}</p>
            {% endfor %}
            {% endif %}
            {% endwith %}
            {{ wtf.quick_form(form, novalidate=True, button_map={"submit": "primary"}) }}
        </div>
    </div>
</div>
```
4. /login 경로에서 사용자의 이메일이 데이터베이스에 존재하지 않거나 비밀번호가 check_password()를 사용하여 저장된 것과 일치하지 않을 경우, 다시 /login으로 리디렉션하고, 사용자에게 어떤 문제가 발생했는지 알리고 다시 시도하도록 요청하는 플래시 메시지가 표시되도록 해야 합니다.
```python
@app.route('/login', methods=["GET", "POST"])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        user = User.query.filter_by(email=form.email.data).first()
        if user is not None:
            flash("invalid email!")
            return redirect(url_for('login'))

        elif not check_password_hash(user.password, form.password.data):
            flash("password not correct")
            return redirect(url_for('login'))

        else:
            login_user(user)
            return redirect(url_for('get_all_posts'))

    return render_template("login.html", form=form)
```

5. 어떻게 내비게이션 바를 업데이트하면 사용자가 로그인하지 않은 경우, 단, 사용자가 등록 후 로그인/인증된 경우에는 내비게이션 바에 다르게 표시되어야 합니다.
```python
#header.html
        {% if not current_user.is_authenticated: %}
          <li class="nav-item">
            <a class="nav-link" href="{{ url_for('login') }}">Login</a>
          </li>
          <li class="nav-item">
            <a class="nav-link" href="{{ url_for('register') }}">Register</a>
          </li>
        {% else: %}
          <li class="nav-item">
            <a class="nav-link" href="{{ url_for('logout') }}">Log Out</a>
          </li>
         {% endif %}
```

6. 사용자가 로그아웃 버튼을 클릭하면 로그아웃하고 홈페이지로 다시 이동하도록 /logout 경로를 코딩하세요.

다음과 같이 되어야 합니다.
```python
@app.route('/logout')
def logout():
    logout_user()
    return redirect(url_for('get_all_posts'))
```
### 요구 사항 3 - 경로 보호
우리가 만든 블로그에서는 최초로 등록된 사용자가 관리자가 됩니다. 관리자는 새 블로그 게시물을 생성할 수 있고, 게시물 수정 및 삭제도 할 수 있습니다.

1. 최초 사용자의  id 는 1입니다. 여기서 index.html 파일과 post.html 파일을 수정해서 관리자에게만 “새 게시물 생성”과 게시물 “수정” 및 “삭제” 버튼이 보이게 할 수 있습니다.
```python
#index.html
<!--        If user id is 1 then they can see the delete button -->
            {% if current_user.id == 1: %}
            <a href="{{url_for('delete_post', post_id=post.id) }}">✘</a>
            {% endif %}
<!--    If user id is 1 then they can see the Create New Post button -->
        {% if current_user.id == 1: %}
        <div class="clearfix">
          <a class="btn btn-primary float-right" href="{{url_for('add_new_post')}}">Create New Post</a>
        </div>
        {% endif %}
```
```python
#post.html
<!--           If user id is 1 then they can see the Edit Post button -->
          {% if current_user.id == 1 %}
           <div class="clearfix">
          <a class="btn btn-primary float-right" href="{{url_for('edit_post', post_id=post.id)}}">Edit Post</a>
          </div>
          {% endif %}
```

2. 사용자에게 해당 버튼이 보이진 않지만, 수동으로 /edit-post, /new-post 및 /delete 라우트에 액세스할 수 있습니다.  @admin_only라는 Python 데코레이터를 생성해서 해당 라우트를 보호하십시오.

액세스를 시도하는 현재 사용자의 id가 1이라면 해당 라우트에 액세스할 수 있지만, id가 1이 아니라면 403 오류(권한 없음)를 받습니다.
```python
def admin_only(func):
    @wraps(func)
    def dec_function(*args, **kwargs):
        if current_user.id == 1:
            return func(*args, **kwargs)
        else:
            return abort(403)
    return dec_function

#Mark with decorator
@admin_only
def add_new_post():
#...
@app.route("/edit-post/<int:post_id>", methods=["GET", "POST"])
@admin_only
def edit_post(post_id):
#...
@app.route("/delete/<int:post_id>")
@admin_only
def delete_post(post_id):
#...
```

### 관계형 데이터베이스 만들기
최초의 사용자가 관리자이며 블로그 소유자가 됩니다. 사용자가 생성한 게시물을 해당 사용자와 데이터베이스에서 연결되도록 합니다. 추후 다른 사용자가 블로그에 게시물을 생성할 수 있도록 초대하고 관리자 권한을 주어야 할 수도 있습니다.

따라서  User 테이블과  BlogPost 테이블 간의 관계를 생성하고 연결해야 합니다. 관계로 연결해야 사용자가 생성한 블로그 게시물을 볼 수 있습니다. 혹은 특정 블로그 게시물의 작성자가 어느 사용자인지 확인할 수 있죠.

단순히 Python 코드만 작성한다면, BlogPost 객체 목록이 들어있는 posts 라는 프로퍼티를 가진  User 객체를 생성한다고 생각하면 됩니다.

예를 들면,

```
class User:
    def __init__(self, name, email, password):
         self.name = name
         self.email = email
         self.password = password
         self.posts = []
 
class BlogPost:
    def __init__(self, title, subtitle, body):
         self.title = title
         self.subtitle = subtitle
         self.body = body
 
new_user = User(
    name="Angela",
    email="angela@email.com",
    password=123456,
    posts=[
        BlogPost(
            title="Life of Cactus",
            subtitle="So Interesting",
            body="blah blah"
        )
    ]        
}
```


이를 통해 특정 사용자가 작성한 모든 게시물을 쉽게 찾을 수 있습니다. 그 반대의 경우는 어떻게 할까요? 특정 게시물 객체를 작성한 사용자는 어떻게 찾을까요? 이를 위해, 단순한 Python 데이터 구조가 아닌, 데이터베이스를 사용하는 것입니다.

SQLite, MySQL 혹은 Postgresql 등의 관계형 데이터베이스에서는 ForeignKey 메서드와 relationship() 메서드를 사용해서 테이블 간 관계를 정의할 수 있습니다.

예를 들어 사용자가 여러 개의 게시물 객체를 생성할 수 있는 상황에서, 사용자 테이블과 게시물 테이블 간에 일대다 관계를 생성하려면 SQLAlchemy 공식 문서를 참고하세요.


과제 1: 부모에 해당하는 사용자와 자식에 해당하는 게시물 클래스 코드를 수정하여 두 테이블 간에 양방향 일대다 관계를 생성해 보세요. 사용자가 생성한 게시물과 게시물 객체의 사용자를 쉽게 찾을 수 있을 것입니다.

```python
from typing import List
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship, Mapped, mapped_column


class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(100), unique=True)
    password = db.Column(db.String(100))
    name = db.Column(db.String(100))
    posts:Mapped[List["BlogPost"]] = relationship(back_populates="author")


class BlogPost(db.Model):
    __tablename__ = "blog_posts"
    id = db.Column(db.Integer, primary_key=True)
    author = db.Column(db.String(250), nullable=False)
    title = db.Column(db.String(250), unique=True, nullable=False)
    subtitle = db.Column(db.String(250), nullable=False)
    date = db.Column(db.String(250), nullable=False)
    body = db.Column(db.Text, nullable=False)
    img_url = db.Column(db.String(250), nullable=False)
    author_id: Mapped[int] = mapped_column(ForeignKey("user.id"))
    author: Mapped["User"] = relationship(back_populates="posts")
```

스키마 수정 후 데이터베이스 재생성
이 상황에 블로그를 재실행한다면 오류가 발생할 것입니다.

OperationalError: (sqlite3.OperationalError) no such column: blog_posts.author_id

main.py 파일의 새 코드가 시작 코드의 원본 데이터베이스 파일인 blog.db  에는 존재하지 않았던 새로운 열을 추가함으로써 데이터베이스를 수정하기 때문입니다.

author_id = db.Column(db.Integer, db.ForeignKey("users.id"))
이렇게 되면 보존할 가치가 있는 데이터가 사라지며, 기존 blog.db 파일을 완전히 삭제하고 db.create_all() 라인을 사용해 테이블을 처음부터 재생성하는 것이 가장 쉽습니다. 단, 이 방법은 데이터베이스를 완전히 정리하는 것이므로 사용자를 다시 등록하고 게시물도 다시 생성해야 합니다.

블로그 웹사이트를 새로 고치면, index.html 페이지와 page.html 페이지에서 작성자 이름이 사라질 것입니다.

과제 2: index.html 페이지와 post.html 페이지를 수정하여 적절한 위치에 작성자 이름이 출력해 보세요.
```python
#index.html
<a href="#">{{post.author.name}}</a>
#post.html
<a href="#">{{post.author.name}}</a>
#post.author 에서 post.author.name 으로 바뀜.
```
힌트: BlogPost 의 author 프로퍼티는 User 객체가 됩니다.


### 요구 사항 4 - 모든 사용자가 블로그 게시물에 댓글을 추가할 수 있도록 하기
1. form.py 파일에 CommentForm을 생성하면 사용자가 댓글을 작성할 수 있는 CKEditorField가 단 하나만 생길 것입니다.
```python
#forms.py
class CommentForm(FlaskForm):
    comment_text = CKEditorField("Comment", validators=[DataRequired()])
    submit = SubmitField("Submit Comment")

#post.html
          #Load the CKEditor
            {{ ckeditor.load() }}
          #Configure it with the name of the form field from CommentForm
            {{ ckeditor.config(name='comment_text') }}
          #Create the wtf quickform from CommentForm
            {{ wtf.quick_form(form, novalidate=True, button_map={"submit": "primary"}) }}
```

다음 단계는 사용자가 댓글을 작성하고 저장할 수 있도록 하는 것입니다. 데이터베이스 내 테이블 간에 관계를 어떻게 설정하는지 배웠습니다. 어느 사용자든 게시물에 댓글을 작성할 수 있는 새로운 테이블을 생성해 관계를 업그레이드해 봅시다.

2. tablename이  "comments"인  Comment(댓글) 테이블을 만드세요. id와 text프로퍼티를 넣어서 CKEditor에 기본 키와 텍스트가 들어가도록 하십시오.
```python
class Comment(db.Model):
    __tablename__ = "comments"
    id = db.Column(db.Integer, primary_key=True)
    text = db.Column(db.Text, nullable=False)
```

3. 부모에 해당하는 User 테이블과 자식에 해당하는 Comment 테이블 간에 일대다 관계를 설정하십시오. 한 User가 다수의 Comment 객체에 연결될 것입니다.
```python
class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(100), unique=True)
    password = db.Column(db.String(100))
    name = db.Column(db.String(100))
    posts: Mapped[List["BlogPost"]] = relationship(back_populates="author")
    comments: Mapped[List["Comment"]] = relationship(back_populates="comment_author")

class Comment(db.Model):
    __tablename__ = "comments"
    id = db.Column(db.Integer, primary_key=True)
    text = db.Column(db.Text, nullable=False)
    author_id: Mapped[int] = mapped_column(ForeignKey("user.id"))
    comment_author: Mapped["User"] = relationship(back_populates="comments")
```

4. 부모에 해당하는 각 BlogPost객체와 자식에 해당하는 Comment객체 간에 일대다 관계를 설정하십시오. 각 BlogPost에 관련된 Comment객체가 여러 개 있을 수 있습니다.
```python
class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(100), unique=True)
    password = db.Column(db.String(100))
    name = db.Column(db.String(100))
    posts: Mapped[List["BlogPost"]] = relationship(back_populates="author")
    comments: Mapped[List["Comment"]] = relationship(back_populates="comment_author")

class BlogPost(db.Model):
    __tablename__ = "blog_posts"
    id = db.Column(db.Integer, primary_key=True)
    author = db.Column(db.String(250), nullable=False)
    title = db.Column(db.String(250), unique=True, nullable=False)
    subtitle = db.Column(db.String(250), nullable=False)
    date = db.Column(db.String(250), nullable=False)
    body = db.Column(db.Text, nullable=False)
    img_url = db.Column(db.String(250), nullable=False)
    author_id: Mapped[int] = mapped_column(ForeignKey("user.id"))
    author: Mapped["User"] = relationship(back_populates="posts")
    comments: Mapped[List["Comment"]] = relationship(back_populates="parent_post")

class Comment(db.Model):
    __tablename__ = "comments"
    id = db.Column(db.Integer, primary_key=True)
    text = db.Column(db.Text, nullable=False)
    author_id: Mapped[int] = mapped_column(ForeignKey("user.id"))
    comment_author: Mapped["User"] = relationship(back_populates="comments")
    post_id: Mapped[int] = mapped_column(ForeignKey("blog_posts.id"))
    parent_post: Mapped["BlogPost"] = relationship(back_populates="comments")
```

5. 이렇게 새 테이블을 추가하면, 기존의 blog.db 파일을 완전히 삭제하고 db.create_all() 라인을 이용해 모든 테이블을 새로 생성하는 것이 좋습니다.

단, 새로운 관리자(id == 1)와 새 게시물 그리고 댓글을 작성할 다른 사용자를 생성해야 합니다.


6. 사용자 ‘아무개’로(혹은 관리자가 아닌 다른 사용자로서) 로그인해 게시물에 댓글을 작성해 보세요. 이를 위해서는 /post/<int:post_id> 라우트를 업데이트해야 합니다. 인증된(로그인된) 사용자만 댓글을 저장할 수 있도록 주의하십시오. 로그인하지 않았다면, 로그인하라는 메시지를 띄우고 /login로 리디렉션하십시오.
```python
@app.route("/post/<int:post_id>", methods=["GET", "POST"])
def show_post(post_id):
    form = CommentForm()
    requested_post = BlogPost.query.get(post_id)

    if form.validate_on_submit():
        if not current_user.is_authenticated:
            flash("You need to login or register to comment.")
            return redirect(url_for("login"))

        new_comment = Comment(
            text=form.comment_text.data,
            comment_author=current_user,
            parent_post=requested_post
        )
        db.session.add(new_comment)
        db.session.commit()

    return render_template("post.html", post=requested_post, form=form, current_user=current_user)
```

7. post.html 파일의 코드를 업데이트해 게시물에 관련된 모든 댓글을 출력하십시오.
```python
#post.html
                <div class="col-lg-8 col-md-10 mx-auto comment">
                    {% for comment in post.comments: %}
                    <ul class="commentList">
                        <li>
                            <div class="commenterImage">
                                <img src="https://pbs.twimg.com/profile_images/744849215675838464/IH0FNIXk.jpg"/>
                            </div>
                            <div class="commentText">
                                {{comment.text|safe}}
                                <span class="date sub-text">{{comment.comment_author.name}}</span>

                            </div>
                        </li>
                    </ul>
                    {% endfor %}
                </div>
```

Gravatar 이미지는 인터넷상 어디에서든 댓글 작성자 이미지로서 쓰입니다.

예를 들면, 아래 게시물의 댓글에서 보실 수 있습니다:


블로그 웹사이트 곳곳에서 사용하는 Gravatar 이미지를 다음 웹사이트에서 바꿀 수 있습니다. http://en.gravatar.com/

Flask 애플리케이션에 Gravatar 이미지를 구현하는 건 아주 간단합니다.

7. 여기서 Gravatar 공식 문서를 보고 댓글 섹션에 Gravatar 이미지를 추가하십시오.
```python
#main.py
from flask_gravatar import Gravatar

gravatar = Gravatar(app, size=100, rating='g', default='retro', force_default=False, force_lower=False, use_ssl=False, base_url=None)
```
```python
#post.html
 <div class="commenterImage">
     <img src="{{ comment.comment_author.email | gravatar }}"/>
</div>
```

## 참고 문서
- https://flask-login.readthedocs.io/en/latest/#login-example
- sqlalchemy 일대다 문서 
- https://docs.sqlalchemy.org/en/20/orm/basic_relationships.html#basic-relationship-patterns
- Blog Post Model Hierarchy
- https://github.com/SadSack963/day-69_blog_with_users/blob/master/docs/Class_Diagram.png
- Gravatar 문서(댓글 이미지)
- http://en.gravatar.com/site/implement/images
