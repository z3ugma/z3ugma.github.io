---
title: A Homegrown Recipe App That Helps You Cook
date: 2015-01-17 04:50:55 Z
layout: post
comments: true
---

I do love to cook. All of my favorite recipes are stored in my head, are improvised off the cuff, or found in a gross stack of crinkled papers held together with a binder clip in my cabinet.

I do a lot of looking up recipes online, but every recipe website I've seen has a big flaw. These sites are not designed to help you actually read the recipe. They are designed to keep you clicking around, keep you sharing recipes on social media, keep you seeing ads.

I wanted a recipe app that would just show me what I needed to gather and how to put it together to cook it all. In the meantime, I decided it wouldn't hurt it if were also pretty to look at. Lastly, I wanted it optimized for use on an iPad so that I could keep it close at hand while actually cooking.

![Finished Recipes Webapp](https://raw.githubusercontent.com/z3ugma/rethink-recipes/master/screenshot1.png)

## Technical Design ##

Python is my pet language, and I wanted a database that would be easy to use with native Python types. Recipes are well-suited to a document-oriented store - each recipe has highly-variable ingredients. The only potential advantage I could see for a relational db would be to answer queries like "show me all the recipes with onion as an ingredient". 
[RethinkDB](http://www.rethinkdb.com) is great because it has a native web interface and a [cool query language](http://www.rethinkdb.com/docs/introduction-to-reql/) called ReQL.

I used Python and Flask to write the app, and Tornado to make it easy to deploy - I can never seem to get uWSGI or Gunicorn working correctly.

## Frontend Design ##

Two design elements set Rethink Recipes apart:

1. When a recipe is entered, the app finds the top 8 Google Images hits for the title of the recipe. It uses the colors found in these images to calculate a __unique, indivualized color palette__ for each recipe.

2. Whenever an ingredient appears verbatim in the directions, it is __highlighted__ in the directions. This makes it easy to scan the directions for the step you should be on next.

## App Routes ##

Here are the nuts and bolts of how Rethink Recipes works:

### Adding a Recipe ###

![Adding a Recipe](https://raw.githubusercontent.com/z3ugma/rethink-recipes/master/screenshot3.png)

When accessing <code>/add</code>, the user is presented with 3 form fields - the title, ingredients, and directions.

When the form is submitted, a bunch of things happen:

1. The title of the recipe is joined with + signs. i.e. "Foo bar baz pancakes" -> "foo+bar+baz+pancakes". This is the format the Google Images api receives queries in.
2. A Google Image API query returns a list of URLs pointing to Google Images thumbnails.
3. A Python module called Colorific, which uses PIL (the Python Image Library), extracts the most prominent colors from each image, 5 colors per image. (I also wrote my own module to do this processing, but Colorific was already packaged and easy to install. )
4. We're left with a list of RGB tuples. 5 colors per images times 8 images = 40 tuples long.
5. The Lightness value of each tuple is calculated (conversion to HSV color space), and the list is reordered in order of darkest to lightest.
6. The list is split into 4 smaller lists of 10 elements each. This roughly groups the colors into the 10 darkest, 10 next darkest, etc.
7. The average value of the 10 RGB tuples in each list is calculated. We end up with 4 tuples to store to the db along with the recipe.
8. The URL title is "slugified" - stripping out spaces and non alphanumeric punctuation to make a nice URL slug
9. The ingredients list is calculated. For each line in the ingredients box:
 - Split on spaces. The first 2 pieces are the amount - "1 cup" for example.
 - Everything after the amount is the "what"
10. The directions list is each line of input in the dirctions box
11. All this data is formed into a Python dictionary:
 - Title, URLs of image thumbs, ingredients, directions, slug, and average colors.
12. Ingredients and directions are sanitized for disallowed entries (like null string)
13. This Python dict is inserted into RethinkDB.
14. The user is redirected to the newly-created recipe.

{% highlight python %}

def get_gimages(query):
    answer = []
    for i in range(0,8,4):
        url = 'https://ajax.googleapis.com/ajax/services/search/images?v=1.0&q=%s&start=%d&imgsz=large&imgtype=photo' % (query,i)
        r = urllib2.urlopen(url)
        for k in json.loads(r.read())['responseData']['results']:
             answer.append(k['tbUrl'])
    return answer


@app.route('/add', methods = ['GET', 'POST'])
def add():
    form = RecipeForm()
    #form.ingredients.data = "1c ingredient\nenter ingredients here  "

    if request.method == 'POST' and form.validate():

        lightness = []
        rainbow = []
        resp=[]
        urls = get_gimages("+".join(form.title.data.split()))
        for url in urls:
            i = cStringIO.StringIO(urllib.urlopen(url).read())
            quiche = colorific.extract_colors(i, max_colors=5)
            resp.extend([each.value for each in quiche.colors])
        resp = [(m,sqrt(0.299 * m[0]**2 + 0.587 * m[1]**2 + 0.114 * m[2]**2)) for m in resp]
        lightness = sorted(resp,key=lambda x: x[1])
        lightness = [i[0] for i in lightness]
        lightness.sort(key=lambda tup: colorsys.rgb_to_hsv(tup[0],tup[1],tup[2])[2])
        for each in chunks(lightness,10):
            avg = tuple(map(lambda y: sum(y) / len(y), zip(*each)))
            rainbow.append(avg)

        slug = slugify(form.title.data)

        recipe = { 'title': form.title.data.title(), 'ingredients': [{'amount': " ".join(ingredient.split()[0:2]), 'what': " ".join(ingredient.split()[2:])} for ingredient in form.ingredients.data.split('\r\n')], 'directions': form.directions.data.split('\r\n'), 'urls': urls, 'slug': slug, 'avgcolors': [list(i) for i in rainbow]}

        recipe['ingredients'] = [i for i in recipe['ingredients'] if not i['what']=='']
        recipe['directions'] = [i for i in recipe['directions'] if not i=='']

        r.table('recipes').insert(recipe).run(g.rdb_conn)

        return redirect(url_for('recipe', query=slug))

    return render_template("add.html", form=form)
{% endhighlight %}

### Editing a Recipe ###
The same form from /add is rendered, with the data from the existing recipe filled in. Once the form is submitted, the same process as adding happens. Instead of inserting the new recipe, however, the existing recipe's record is updated.

The slug of the recipe appears in the URL of the page. We use that slug to get the first recipe in the db that matches that slug. Note that there's no guarantee of uniqueness for a slug - RethinkDB doesn't have unique indexing.

{% highlight python %}

@app.route('/<query>/edit', methods = ['GET', 'POST'])
def edit(query):


    form = RecipeForm()
    recipe = list(r.table('recipes').filter({'slug': query}).run(g.rdb_conn))

    if recipe:
        recipe = recipe[0]
    else:
        abort(404)

    if request.method == 'GET':

        form.ingredients.data = "\r\n".join([(i['amount'] + " " + i['what']) for i in recipe['ingredients']])
        form.directions.data = "\r\n".join(i for i in recipe['directions'])
        form.title.data = recipe['title']

        return render_template("edit.html", form=form, recipe=recipe)

{% endhighlight %}

{% highlight python %}

        id = recipe['id']

        r.table('recipes').get(id).update(recipe).run(g.rdb_conn)

        return redirect(url_for('recipe', query=slug))

{% endhighlight %}

### Deleting a Recipe ###

![Deleting a Recipe](https://raw.githubusercontent.com/z3ugma/rethink-recipes/master/screenshot4.png)

The recipe's slug is in the URL. The user clicks "Delete", is prompted with a checkbox to confirm, and then recipes with slugs matching the URL are deleted:

{% highlight python %}

@app.route('/<query>/delete', methods = ['GET', 'POST'])
def delete(query):
    form = DeleteForm()
    recipe = list(r.table('recipes').filter({'slug': query}).run(g.rdb_conn))
    if recipe:
        recipe = recipe[0]
    else:
        abort(404)

    if request.method == 'GET':

        return render_template("delete.html", query=query, recipe=recipe, form=form)

    if request.method == 'POST':
        if 'deleterecipe' in request.form:

            id = recipe['id']

            r.table('recipes').get(id).delete().run(g.rdb_conn)
            return redirect(url_for('index'))
        else:
            flash("Are you sure? Check the box")
            return render_template("delete.html", query=query, recipe=recipe, form=form)
{% endhighlight %}

### Displaying a Recipe ###

When a recipe's slug appears alone in the URL, the recipe is displayed.

The first recipe in the db with a matching slug is displayed. We take the "what" from each ingredient, and only take the portion before a comma. This allows for things like "onions, diced" to appear in the recipe, but still highlight the word "onions" in the directions. If the word appears in the directions, then the word in the directions is wrapped in kbd tags, which stand out in Bootstrap.

You'll recall that each recipe has 8 URLs of Google images and 4 RBG tuples of colors to use in displaying the page. When the page is rendered, the darkest of the RGB colors is used as the background of the header, the lightest is used for the page background and header text, and the 2nd darkest is used for the panel headers. The 4th one is reserved for future use.

Along with the 8 Google images thumbnails in place, this makes each recipe page have the unique colors corresponding to the food you're making! So a raspberry pie has bright reds and rich tans, and a chili recipe has lots of earthy tones.

{% highlight python %}

@app.route('/<query>')
def recipe(query):
    recipe = list(r.table('recipes').filter({'slug': query}).run(g.rdb_conn))

    if recipe:
        recipe = recipe[0]
    else:
        abort(404)

    steps = []
    for i in recipe['directions']:
        for j in [k['what'].split(",")[0] for k in recipe['ingredients']]:
            i = i.replace(j, ("<kbd>" + j + "</kbd>"))
        steps.append(Markup(i))

    rainbow = [tuple(l) for l in recipe['avgcolors']]

    return render_template("recipes.html", urls = recipe['urls'], avg=rainbow, ingredients = recipe['ingredients'], steps = steps, title=recipe['title'], query=query)


{% endhighlight %}

The end result of this is an iPad-optimized recipe database, customizable for your own personal recipes. Not great for importing directly from recipe sites, but great for saving your time-tested recipes in a user-friendly format.

### Futures

I would like to expand this to being multi-user. I think each user would get their own RethinkDB table, and add a users table with hashed passwords.

### Check It Out
You can download and set it up for yourself in a virtual machine - see the [Github repo for the project](https://github.com/z3ugma/rethink-recipes)

