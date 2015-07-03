# ExecJS Unexpected token: punc (:) 

This is how I fixed it.
Modified ExecJS::ExternalRuntime#exec to not unlink file in case of error

```
result = extract_result(@runtime.exec_runtime(filepath), filepath)
File.unlink(tmpfile)
result
```

So I had now the file with error, something like execjs20150702-17011-16105vfjs.

I run node

```
node execjs20150702-17011-16105vfjs
```

Go error

```
print(JSON.stringify(['err', '' + err, err.stack]));
^
ReferenceError: print is not defined
```
I changed code to use console.log instead of print. I ran node again and I got error:

```
Unexpected token: punc (:) (line: 38959, col: 7, pos: 1234839)
```

Then I modified execjs file to dump source that it parses, I added such code after var source="very long js source":

```
var fs = require('fs');
fs.writeFile('source.js', source, function (err) {
  if (err) throw err;
  console.log('It\'s saved!');
});
```

I opened source.js and went to line 38959 and I saw some bad js:

```
},{"./emptyFunction":107}]},{},[1])(1)
}); 18: 'Alt',
  19: 'Pause',
  20: 'CapsLock',
  27: 'Escape',
  32: ' ',
  33: 'PageUp',
```

It seems like some file were rewrited. I used grep to find what js library have such code and I found that react-source that is used by react-rails gem.

I investigated source of react-rails gem to check what it does. I found very interesting line in ```react-rails-0.12.1.0/lib/react/rails/railtie.rb```

```
tmp_path = app.root.join('tmp/react-rails')
filename = 'react' + (addons ? '-with-addons' : '') + (variant == :production ? '.min.js' : '.js')
FileUtils.mkdir_p(tmp_path)
FileUtils.cp(::React::Source.bundled_path_for(filename),
             tmp_path.join('react.js'))
FileUtils.cp(::React::Source.bundled_path_for('JSXTransformer.js'),
             tmp_path.join('JSXTransformer.js'))
```

Wait, I start my environment with zeus and it forks process and run development and test environment in parallel, so here can be the problem. I modified tmp_path initialization to look like:

```
tmp_path = app.root.join('tmp/react-rails', ::Rails.env)
```

Started zeus again with ```zeus start``` and I saw it created two folders:
development and test. When I checked react.js file size, it was different in environments:

```
-rw-r--r--+ 1 lz  staff  571784  3 Iul 12:07 ./development/react.js
-rw-r--r--+ 1 lz  staff  625422  3 Iul 12:07 ./test/react.js
```
So this was the problem when js file from test environment overrided smaller one from development environment.

I forked github problem and fixed gem in branch 0.12 because it will take some time while our team will upgrade to newer gem version.

Here is the [commit with the fix][1].

[1]: https://github.com/Assembla/react-rails/commit/ae9edc29cc641c825fcea14efc5efa06f3f9f1dd
