require 'rake'
require 'bundler/setup'
require 'logger'
Bundler.require(:default)
USERS = 20000

task :setup do
  puts "Setup"
  # 10 million users
  # each user has 200 friends (unchanging)
  # we'll have 10000 users connected (this will change)
  # 10,000 friend requests per second. can it do it?
  redis = Redis.new
  USERS.times do |user_id|
    200.times do |friend_offset|
      friend_id = user_id + friend_offset + 1
      redis.sadd("user:#{user_id}", friend_id.to_s)
    end
  end
end

task :teardown do
  puts "Tearing down"
  redis = Redis.new
  redis.keys.each do |key|
    redis.del(key)
  end
end

task :test, :port do |t, args|
  success = 0
  error = 0
  port = (args[:port] ||  "6379").to_s
  puts port

  redis = Redis.new
  if true #port == "6379"
    redis.del("users_online") 
    100.times do
      index = rand(USERS)
      10.times do |offset|
        redis.sadd("users_online", (index + offset))
      end
    end
    100000.times do |i|
      id = USERS + i
      redis.sadd("users_online", id)
    end
  end
  logger = Logger.new("shit.log")

  online = nil
  EM.run do
    puts "Starting #{port}"
    redis = EM::Hiredis.connect("redis://127.0.0.1:#{port}/0")
    master = EM::Hiredis.connect
    # we imagine users are coming online/offline sporadically
    EM::PeriodicTimer.new(1.0/1000) do
      defer = master.spop("users_online")
      defer.callback { success += 1 }
      defer.errback { error += 1 }
    end
    EM::PeriodicTimer.new(1.0/1000) do
      user_id = rand(USERS)
      7.times do |offset|
        defer = redis.sadd("users_online", user_id + offset)
        defer.callback {|a| success += 1 }
        defer.errback { |a| error += 1 }

        defer = redis.sinter("users_online", "user:#{user_id}")
        #defer.callback {|a| logger.info a; success += 1 }
        defer.callback {|a| success += 1 }
        defer.errback { |a| error += 1 }
      end
    end

    EM::PeriodicTimer.new(1) do
      redis.smembers("users_online").callback do |result|
        online = result.count
        puts "Online: #{online}"
      end
    end

    EM::Timer.new(30) do
      puts "Success: #{success}"
      puts "Error: #{error}"
      puts "Rate: #{(success + error).to_f/30}"
      puts "Online: #{online}"
      EM.stop
    end
  end
end
