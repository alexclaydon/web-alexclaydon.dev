I've recently been spending a lot of time reading, thinking about and trying to understand different architectural paradigms in software design - specifically in Python - as I try to alight on one that makes sense to me personally. That is probably for another (long) post, but a 2018 [discussion](https://news.ycombinator.com/item?id=18043058) on hackernews of [this](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell) article, which also led me to [this](https://www.youtube.com/watch?v=DJtef410XaM) presentation by Brandon Rhodes - a big name in the Python space - all got me thinking about a particular problem I found that I was having when trying to write code in Python: how best to handle file IO operations.

My understanding of the take away from the discussion and sources is that one of the nice things about following a "functional core, imperative shell" paradigm is that you can place all of your file IO operations on the outside of your code - in the "imperative shell", so to speak - and find that as a result you have highly readable code with namespacing that naturally makes sense.

As a quick aside, I found loads of seemlingly irrelevant or outdated resources out there for file IO in Python, but [this](https://stackoverflow.com/questions/3430372/how-do-i-get-the-full-path-of-the-current-files-directory) was the best, most current one that I could find.

## File IO in Python

Note that we are talking here about reading from (as input), or writing to (as output), _files_, not packages or modules (which are 'visible' to Python by way of a unique name by which they can be imported). I will look at good practices for package / module imports separately at a later date.

Let's take as our starting point a simple function that either writes to or reads from a file. On the face of it, the first question is whether you expect to be using that function directly in the package in which it resides, or importing it from somewhere else. In other words, is the function part of a library, or an app? (However, we'll see in due course that the distinction is less important if you stick to "pure" functions and a "functional core, imperative shell" approach.)

For a library, by definition very often the file writing or reading functions are being imported into an app elsewhere and performing their input / output there. So, if it's part of a library, keep your path state information out of the function altogether. That is, keep your functions as pure as possible, with as few side-effects as possible.

```python
from pathlib import Path

contents = ['some-content', 'some other content']


def file_writer(target: Path):
    if not target.exists():
        target.touch()
    with open(target.as_posix(), mode="w") as outfile:
        for line in contents:
            outfile.write(f"{line}\n\n")
```

On the other hand, if the function is part of an app or normal package and needs to place a file in a specific location - for example, to be later read by another function - or read the file from a specific location, then you have to pass a path to that function somehow - either by specifying a default path in the function itself, or by passing a path into the function when it's called.

Here, too, let's keep the path information out of the function body itself, and take a "functional core, imperative shell" approach: in any given module, keep your functions pure but then set up and pass your path variables in your **main**() function.

```python
from pathlib import Path

contents = ['some-content', 'some other content']


def file_writer(target: Path):
    if not target.exists():
        target.touch()
    with open(target.as_posix(), mode="w") as outfile:
        for line in contents:
            outfile.write(f"{line}\n\n")


if __name__ == '__main__':
    TARGET_FILE = 'target-file.md'
    TARGET_PATH = (Path.cwd() / 'resources')
    file_writer(target=TARGET_PATH / TARGET_FILE)
```

A question arises here then: how do you describe the path you want?

## Specifying Python paths

Here are the choices:

```python
# relative to working directory (which will change depending on from where the function is called)

TARGET_PATH = Path.cwd() / 'resources'

# relative to file system location of the file containing the called function (which is stable over time)

TARGET_PATH = Path(__file__).parent.absolute()

# absolute

TARGET_PATH = Path.home() / 'dev' / 'projects' / 'pvmon' / 'pvmon' / 'resources'
```

It's unlikely that you want to use an absolute path - unless you're trying to interact with files outside of your package and in a location that is stable over time. I would think this would be more common for reads than for writes.

Using a path relative to the working directory initially looks problematic, because it appears to move around so much - sometimes it refers to the directory containing the file containing the function that you're calling; sometimes it will refer to a parent or sibling directory of that directory (if it's called from some other part of your package). However, on closer inspection the answer is perhaps simpler than it looks: if you're following a "functional core, imperative shell" model, regardless of the file system location of the various functions, you will be calling all of them from the same location: that is, your `main.py`, or `app.py` file (or whatever you call it) - that is: the module that contains your `main()` function:

That being the case, your `Path.cwd()` (and any sub-directories hanging off it) will always refer to the same directory - the directory containing your `main()` function. You can pass it to any of the functions invoked by your `main()` function at runtime and they'll all be dealing with the same directory. Accordingly, you would think that the vast majority of cases would be using this relative model.

What about the final choice - path relative to the file containing the function being called? I honestly can't see a use for this at the moment. If you're following "functional core, imperative shell", it seems like you shouldn't need to do this.

## In conclusion

Keep your functions pure, pass any path parameters to those functions from your "imperative shell", and be sure to correctly specify your paths depending on your use-case.

The [textbase](https://github.com/alexclaydon/textbase) repo on my Github page is an example of me experimenting with a "functional core, imperative shell" paradigm.