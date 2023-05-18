+++
title = 'File rename in ruby'
date = 2014-04-13
tags = ["ruby"]
+++

While doing some exercise of the book [Learn to Program of Chris Pine](https://pine.fm/LearnToProgram/), trying to rename some files
as part of some exercise, that was renaming a bunch of picture from a folder, I noticed a strange functionality.



Using the method `File.rename(old_name, new_name)` and after running my program it seems that the file haven't been rename instead it has been deleted, but that's not what happen.
What ruby was doing is renaming and moving the file to my printing working directory.
What really solve it was giving an absolute path to the method.

`File.rename(old_name, "/Users/gus/Desktop/myImages/new_name")`


And I thought that was some tedious work, I look up in [Ruby doc ](http://www.ruby-doc.org/) and found that `ruby` is smart enough to provide with a method.
`File.join(Dir.pwd, "new_name")` this method will concatenate every string we pass it, providing the path to our file.

This is my final code for the exercise.

```ruby
Dir.chdir  "../../Desktop/Images"
pictures = Dir['*.{jpg,jpeg}']

pictures.each do |picture|
  puts "The #{picture} is inside the folder what do you want to do: 1) rename, 2) delete, 3) nothing"
  answer = gets.chomp.to_i
  if answer == 1
    puts "So you want to rename it. Good what you want to rename it"
    answer = gets.chomp
    File.rename(picture, File.join(Dir.pwd, answer))
    puts "The picture now it is #{answer}"
  elsif answer == 2
    puts "You want to delete it"
    File.delete(picture)
    unless File.exists?(picture)
      puts "The file has been delete it"
    end
  else
    puts "so you want to do nothing to the file"
    puts "Done !!!!"
  end
end

puts "Thanks for handeling our images"
```

