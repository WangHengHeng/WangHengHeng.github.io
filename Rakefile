task :default => :new
require 'fileutils'
desc "创建新 post"
task :new do
  puts "请输入要创建的 post URL："
	@url = STDIN.gets.chomp
	puts "请输入 post 标题："
	@name = STDIN.gets.chomp
	puts "请输入 post 分类，以空格分隔："
	@categories = STDIN.gets.chomp
	@slug = "#{@url}"
	@slug = @slug.downcase.strip.gsub(' ', '-')
	@date = Time.now.strftime("%F")
	@post_name = "_posts/#{@date}-#{@slug}.markdown"
	@author = "王哼哼"
	if File.exist?(@post_name)
			abort("文件名已经存在！创建失败")
	end
	FileUtils.touch(@post_name)
	open(@post_name, 'a') do |file|
			file.puts "---"
			file.puts "layout:     post"
			file.puts "title:      \"#{@name}\""
			file.puts "date:       #{Time.now}"
			file.puts "author:     \"#{@author}\""
			file.puts "categories: #{@categories}"
			file.puts "---"
	end
	exec "vim #{@post_name}"
end