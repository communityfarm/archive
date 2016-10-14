require 'bundler/setup'
require 'dotenv/tasks'
require 'pry'
require 'time'
require 'the_community_farm'
require 'rss'
require 'git'

task build_xml: :dotenv do
  require 'rest-client'
  require 'json'

  # We're always asking for json because it's the easiest to deal with
  morph_api_url = 'https://api.morph.io/communityfarm/organic_boxes_scraper/data.json'

  morph_api_key = ENV.fetch('MORPH_API_KEY')

  response = RestClient.get(
    morph_api_url,
    params: {
      key: morph_api_key,
      query: "select * from 'data'"
    }
  )

  boxes_url = 'http://www.thecommunityfarm.co.uk/boxes/box_display.php'

  scrapes = JSON.parse(response, symbolize_names: true)

  boxes_by_week = scrapes.flat_map do |scrape|
    TheCommunityFarm::OrganicBoxes.new(html: scrape.delete(:html)).map do |box|
      box.to_h.merge(scrape: scrape)
    end
  end

  mkdir_p 'build'

  boxes_by_week.group_by { |box| box[:name] }.each do |box_name, boxes|
    filename = "#{URI.encode_www_form_component(box_name)}.xml"
    rss = RSS::Maker.make('atom') do |maker|
      maker.channel.id = "https://communityfarm.github.io/api/#{filename}"
      maker.channel.author = 'Community Farm'
      maker.channel.updated = boxes.first[:scrape][:created_at]
      maker.channel.title = box_name
      boxes.each do |week|
        maker.items.new_item do |item|
          item.id = "https://communityfarm.github.io/api/#{filename}##{week[:items_sha]}"
          item.link = boxes_url
          item.title = "#{box_name} #{week[:scrape][:created_at]}"
          item.content.content = week[:items].join('<br>')
          item.updated = week[:scrape][:created_at]
        end
      end
    end

    File.write("build/#{filename}", rss.to_s)
  end
end

task build_api: :build_xml

task deploy_api: :build_api do
  git = Git.clone('https://github.com/communityfarm/api', 'communityfarm-api', path: Dir.mktmpdir).tap do |g|
    g.config('user.name', 'Chris Mytton')
    g.config('user.email', 'chrismytton@users.noreply.github.com')
  end
  cp_r 'build/.', git.dir.path
  git.chdir do
    if `git status -s`.chomp.empty?
      warn "No changes to deploy"
      exit
    end
    git.add
    git.commit('Rebuild API')
    git.push(:origin, :master)
  end
end

task default: :deploy_api
