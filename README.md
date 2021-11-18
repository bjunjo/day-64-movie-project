# day-64-movie-project

## Problem: Using HTML, Python, and SQL to make a top 10 movie list website

## Solution
1. Create a database and table

```
# Create a database
app.config['SQLALCHEMY_DATABASE_URI'] = "sqlite:///movies.db"
db = SQLAlchemy(app)

# Create a table
class Movie(db.Model):
    id = db.Column('movie_id', db.Integer, primary_key = True)
    title = db.Column(db.String(250), unique=True, nullable=False)
    year = db.Column(db.Integer, nullable=False)
    description = db.Column(db.String(250), nullable=False)
    rating = db.Column(db.Float, nullable=False)
    ranking = db.Column(db.Integer, nullable=False)
    review = db.Column(db.String(250), nullable=False)
    img_url = db.Column(db.String(250), nullable=False)

def __init__(self, title, year, description, rating, ranking, review, img_url):
    self.title = title
    self.year = year
    self.description = description
    self.rating = rating
    self.ranking = ranking
    self.review = review
    self.img_url = img_url

db.create_all()
```

2. Use WTF quick forms
```
# Forms
class RateMovieForm(FlaskForm):
    rating = StringField(u'Your Rating Out of 10 e.g 7.5', [DataRequired()])
    review = StringField(u'Your Review', [DataRequired()])
    submit = SubmitField(u'Done')

class AddMovieForm(FlaskForm):
    title = StringField(u'Movie Title')
    submit = SubmitField(u'Add Movie')

```

3. Use Flask to create, read, uodatem and delete

```
# Flask starts
# Create
@app.route("/")
def home():
    # Order movies by rating
    movies_order_by_rating = db.session.query(Movie).order_by(Movie.rating)

    # Rank by movies
    movie_total = Movie.query.count()
    for movie in movies_order_by_rating:
        movie_total -= 1
        movie.ranking = movie_total + 1
    return render_template("index.html", movies=movies_order_by_rating)

# Update
@app.route("/edit", methods=['GET', 'POST'])
def edit():
    form = RateMovieForm()

    # Get the corresponding movie
    movie_id = request.args.get('id')
    print(f"Selected Movie ID: {movie_id}")
    movie_selected = Movie.query.get(movie_id)
    print(f"Selected Movie: {movie_selected}")

    # POST update corresponding
    if form.validate_on_submit():
        movie_selected.rating = float(form.rating.data)
        movie_selected.review = form.review.data
        db.session.commit()
        return redirect(url_for('home'))

    return render_template('edit.html', form=form, movie=movie_selected)

# Delete
@app.route("/delete", methods=['GET', 'POST'])
def delete():
    # Delete a movie
    movie_id = request.args.get('id')
    movie_to_delete = Movie.query.get(movie_id)
    db.session.delete(movie_to_delete)
    db.session.commit()
    return redirect(url_for('home'))

# Add
@app.route("/add", methods=['GET', 'POST'])
def add():
    form = AddMovieForm()

    if form.validate_on_submit():
        movies = MOVIE_DATABASE_ENDPOINT + SEARCH_MOVIES
        movie_search_params = {
            'api_key': MOVIE_SEARCH_API_KEY,
            'language': 'en-US',
            'query': request.form['title'],
            'page': 1,
            'include_adult': False
        }

        # Show all the movies available with search query
        response = requests.get(movies, params=movie_search_params)
        response.raise_for_status()
        movie_search_data = response.json()['results']

        return render_template('select.html', movies=movie_search_data, movie_api_key=MOVIE_SEARCH_API_KEY)
    return render_template('add.html', form=form)


@app.route("/find", methods=['GET', 'POST'])
def find_movie():
    movie_id = request.args.get('id')

    # If there's any movie ID,
    if movie_id:
    # Find the movie and print out
        movie_info = MOVIE_DATABASE_ENDPOINT + GET_ONE_MOVIE + movie_id
        movie_info_search_params = {
            'api_key': MOVIE_SEARCH_API_KEY,
            'language': 'en-US'
        }
    # Show the movie's details
    response = requests.get(movie_info, params=movie_info_search_params)
    response.raise_for_status()
    movie_info = response.json()
    print(f"Movie Title Detailed Info: \n {movie_info}")

    # TODO: Adding a new movie to the database
    # Show it to the home page
    new_movie = Movie(
        title=movie_info['original_title'],
        year=movie_info['release_date'],
        description=movie_info['overview'],
        rating=0,
        ranking=0,
        review="Review Missing",
        img_url=f"https://image.tmdb.org/t/p/original/{ movie_info['poster_path'] }"
    )
    db.session.add(new_movie)
    db.session.commit()
    return redirect(url_for('edit', id=new_movie.id))

if __name__ == '__main__':
    app.run(debug=True)
```
