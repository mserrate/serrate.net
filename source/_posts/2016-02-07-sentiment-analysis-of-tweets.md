---
title: Sentiment analysis of tweets
date: 2016-02-07T16:40:34+00:00
thumbnail: /files/2016/02/TwitterTopology_Analysis.png
categories:
  - Architecture
tags:
  - BigData
  - Storm
---
In the previous <a href="http://www.serrate.net/2016/01/05/analysis-of-twitter-streams-with-kafka-and-storm/" target="_blank">post</a> I have presented an overview of the topology used to analyse twitter streams with **Kafka** and **Storm**. Now it&#8217;s time to cover the technical details of the twitter topology.

### Twitter Topology

{% img /files/2016/02/TwitterTopology_Analysis.png 550 254 '"Twitter Streaming"' '"Twitter Streaming"' %}

The declaration of the storm topology using KafkaSpout to read the tweets from a kafka queue:

{% codeblock lang:java %}
public class TwitterProcessorTopology extends BaseTopology {

    public TwitterProcessorTopology(String configFileLocation) throws Exception {
        super(configFileLocation);
    }

    private void configureKafkaSpout(TopologyBuilder topology) {
        BrokerHosts hosts = new ZkHosts(topologyConfig.getProperty("zookeeper.host"));

        SpoutConfig spoutConfig = new SpoutConfig(
                hosts,
                topologyConfig.getProperty("kafka.twitter.raw.topic"),
                topologyConfig.getProperty("kafka.zkRoot"),
                topologyConfig.getProperty("kafka.consumer.group"));
        spoutConfig.scheme= new SchemeAsMultiScheme(new StringScheme());

        KafkaSpout kafkaSpout= new KafkaSpout(spoutConfig);
        topology.setSpout("twitterSpout", kafkaSpout);
    }

    private void configureBolts(TopologyBuilder topology) {
        // filtering
        topology.setBolt("twitterFilter", new TwitterFilterBolt(), 4)
                .shuffleGrouping("twitterSpout");

        // sanitization
        topology.setBolt("textSanitization", new TextSanitizationBolt(), 4)
                .shuffleGrouping("twitterFilter");

        // sentiment analysis
        topology.setBolt("sentimentAnalysis", new SentimentAnalysisBolt(), 4)
                .shuffleGrouping("textSanitization");

        // persist tweets with analysis to Cassandra
        topology.setBolt("sentimentAnalysisToCassandra", new SentimentAnalysisToCassandraBolt(topologyConfig), 4)
                .shuffleGrouping("sentimentAnalysis");

        // divide sentiment by hashtag
        topology.setBolt("hashtagSplitter", new HashtagSplitterBolt(), 4)
                .shuffleGrouping("textSanitization");

        // persist hashtags to Cassandra
        topology.setBolt("hashtagCounter", new HashtagCounterBolt(), 4)
                .fieldsGrouping("hashtagSplitter", new Fields("tweet_hashtag"));

        topology.setBolt("topHashtag", new TopHashtagBolt())
                .globalGrouping("hashtagCounter");

        topology.setBolt("topHashtagToCassandra", new TopHashtagToCassandraBolt(topologyConfig), 4)
                .shuffleGrouping("topHashtag");
    }

    private void buildAndSubmit() throws Exception {
        TopologyBuilder builder = new TopologyBuilder();
        configureKafkaSpout(builder);
        configureBolts(builder);

        Config config = new Config();

        //set producer properties
        Properties props = new Properties();
        props.put("metadata.broker.list", topologyConfig.getProperty("kafka.broker.list"));
        props.put("request.required.acks", "1");
        props.put("serializer.class", "kafka.serializer.StringEncoder");
        config.put(KafkaBolt.KAFKA_BROKER_PROPERTIES, props);

        StormSubmitter.submitTopology("twitter-processor", config, builder.createTopology());
    }

    public static void main(String[] args) throws Exception {
        String configFileLocation = args[0];

        TwitterProcessorTopology topology = new TwitterProcessorTopology(configFileLocation);
        topology.buildAndSubmit();
    }
}
{% endcodeblock %}

<!--more-->

#### 1. Filter Bolt

First of all, we are going to filter the tweets that we are interested in. As we are going to perform the sentiment analysis just to tweets in english, we are filtering on this property:

{% codeblock lang:java %}
public class TwitterFilterBolt extends BaseBasicBolt {
    private static final Logger LOG = LoggerFactory.getLogger(TwitterFilterBolt.class);

    @Override
    public void execute(Tuple tuple, BasicOutputCollector collector) {
        try {
            JSONObject object = (JSONObject)JSONValue.parseWithException(tuple.getString(0));

            if (object.containsKey("lang") &amp;amp;&amp;amp; "en".equals(object.get("lang"))) {
                long id = (long)object.get("id");
                String text = (String)object.get("text");
                String createdAt = (String)object.get("created_at");
                JSONObject entities= (JSONObject)object.get("entities");
                JSONArray hashtags =(JSONArray)entities.get("hashtags");
                HashSet&amp;lt;String&amp;gt; hashtagList = new HashSet&amp;lt;String&amp;gt;();
                for(Object hashtag : hashtags)
                {
                    hashtagList.add(((String)((JSONObject)hashtag).get("text")).toLowerCase());
                }

                collector.emit(new Values(id, text, hashtagList, createdAt));
            }
            else {
                LOG.debug("Ignoring non-english tweets");
            }

        } catch (ParseException e) {
            LOG.error("Error parsing tweet: " + e.getMessage());
        }

    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("tweet_id", "tweet_text", "tweet_hashtags", "tweet_created_at"));
    }
}
{% endcodeblock %}

#### 2. Sanitization Bolt

Then, we will sanitise our tweets by converting accented characters into unaccented characters and by removing single letters or numbers:

{% codeblock lang:java %}
public class TextSanitizationBolt extends BaseBasicBolt {
    private static final Logger LOG = LoggerFactory.getLogger(TextSanitizationBolt.class);

    @Override
    public void execute(Tuple tuple, BasicOutputCollector collector) {

        String text = tuple.getString(1);
        String normalizedText = Normalizer.normalize(text, Normalizer.Form.NFD);
        text = normalizedText.replaceAll("\\p{InCombiningDiacriticalMarks}+", "");
        text = text.replaceAll("[^\\p{L}\\p{Nd}]+", " ").toLowerCase();

        collector.emit(new Values(
                tuple.getLongByField("tweet_id"),
                text,
                tuple.getValueByField("tweet_hashtags"),
                tuple.getStringByField("tweet_created_at")));
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("tweet_id", "tweet_text", "tweet_hashtags", "tweet_created_at"));
    }
}
{% endcodeblock %}

#### 3a. Sentiment Analysis Bolt

On this bolt we are scoring the tweet by each of its words using <a href="http://sentiwordnet.isti.cnr.it/" target="_blank">SentiWordNet</a>. That&#8217;s not the best way to do it as it can have false positives or negatives given that it does the classification word by word independently: it does not cover the tweet context or sarcasm, etc. but that&#8217;s ok for a sample 🙂

{% codeblock lang:java %}
public class SentimentAnalysisBolt extends BaseBasicBolt {
    private static final Logger LOG = LoggerFactory.getLogger(SentimentAnalysisBolt.class);
    SentiWordNet sentiWordNet;

    @Override
    public void prepare(Map stormConf, TopologyContext context) {
        try {
            sentiWordNet = SentiWordNet.getInstance();
        } catch (IOException e) {
            LOG.error("Problem parsing SentiWordNet file: " + e.getMessage());
        }
    }

    @Override
    public void execute(Tuple tuple, BasicOutputCollector collector) {
        double count = 0;
        String text = tuple.getStringByField("tweet_text");

        try {
            String delimiters = "\\W";
            String[] tokens = text.split(delimiters);
            double feeling = 0;
            for (int i = 0; i &amp;lt; tokens.length; ++i) {
                if (!tokens[i].isEmpty()) {
                    // Search as adjective
                    feeling = sentiWordNet.extract(tokens[i], "a");
                    count += feeling;
                }
            }

            LOG.info("text: " + text + " count: " + count);
        }
        catch (Exception e) {
            LOG.error("Problem found when classifying the text: " + e.getMessage());
        }

        collector.emit(new Values(
                tuple.getLongByField("tweet_id"),
                text,
                count,
                tuple.getValueByField("tweet_hashtags"),
                tuple.getStringByField("tweet_created_at")));
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("tweet_id", "tweet_text", "tweet_sentiment", "tweet_hashtags", "tweet_created_at"));
    }
}
{% endcodeblock %}

#### 4a. Save results to Cassandra

Finally, we will store to Cassandra the the tweet with the score and its hashtags if any:

{% codeblock lang:java %}
public class SentimentAnalysisToCassandraBolt extends CassandraBaseBolt {
    private static final Logger LOG = LoggerFactory.getLogger(SentimentAnalysisToCassandraBolt.class);

    public SentimentAnalysisToCassandraBolt(Properties properties) {
        super(properties);
    }

    @Override
    public void execute(Tuple tuple, BasicOutputCollector collector) {

        HashSet<String> hashtags = (HashSet<String>)tuple.getValueByField("tweet_hashtags");

        Statement statement = QueryBuilder.update("tweet_sentiment_analysis")
                .with(QueryBuilder.set("tweet", tuple.getStringByField("tweet_text")))
                .and(QueryBuilder.set("sentiment", tuple.getDoubleByField("tweet_sentiment")))
                .and(QueryBuilder.addAll("hashtags", hashtags))
                .and(QueryBuilder.set("created_at", tuple.getStringByField("tweet_created_at")))
                .where(QueryBuilder.eq("tweet_id", tuple.getLongByField("tweet_id")));

        LOG.debug(statement.toString());

        session.execute(statement);
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
    }
}
{% endcodeblock %}

#### 3b. Hashtag Splitter Bolt

That branch of the topology graph is responsible for splitting the different hashtags and emitting a tuple per hashtag to the next bolt. That&#8217;s why we inherit from BaseRichBolt in order to manually ACK the tuple after all hashtags have been emitted.

{% codeblock lang:java %}
public class HashtagSplitterBolt extends BaseRichBolt {
    OutputCollector collector;
    Map<String, Integer> count = new HashMap<String, Integer>();

    @Override
    public void prepare(Map map, TopologyContext topologyContext, OutputCollector collector) {
        this.collector = collector;
    }

    @Override
    public void execute(Tuple tuple) {
        HashSet<String> hashtags = (HashSet<String>)tuple.getValueByField("tweet_hashtags");

        for (String hashtag : hashtags) {
            collector.emit(new Values(hashtag));
        }

        collector.ack(tuple);
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("tweet_hashtag"));
    }
}
{% endcodeblock %}

#### 4b. Hashtag Counter Bolt

In this particular case, we are using **Fields Grouping** because we want to partition the stream by hashtag. It means that the same hashtag will always go to the same task. Thus, we can use a hashmap to count the number of occurrences of a hashtag:

{% img /files/2016/02/HashtagSplit.png 180 228 '"Hashtag Split"' '"Hashtag Split"' %}

{% codeblock lang:java %}
public class HashtagCounterBolt extends BaseBasicBolt {
    private static final Logger LOG = LoggerFactory.getLogger(HashtagCounterBolt.class);
    private Map<String, Long> hashtag_count = new HashMap<String, Long>();

    @Override
    public void execute(Tuple tuple, BasicOutputCollector collector) {
        String hashtag = tuple.getStringByField("tweet_hashtag");
        Long count = hashtag_count.get(hashtag);

        if (count == null)
            count = 0L;

        count++;
        hashtag_count.put(hashtag, count);

        collector.emit(new Values(hashtag, count));
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("hashtag", "count"));
    }
}
{% endcodeblock %}

#### 5b.  Top Hashtag Bolt

For that bolt, we are using **Global Grouping** to get the top 20 hashtags. Global grouping means that all hashtags will go to the same bolt&#8217;s task. For that use case, we need a sliding windows in order to get the top 20 hashtags every 10 seconds. We are relying on the Storm **Tick Tuple** feature. For normal tuples we just do the ranking of hashtags and, when a tick tuple is received (configured to get it every 10sec) we emit the ranking calculated over this window of time.

{% img /files/2016/02/TopN.png 209 221 '"Top N"' '"Top N"' %}

{% codeblock lang:java %}
public class TopHashtagBolt extends BaseBasicBolt {
    List<List> rankings = new ArrayList<List>();
    private static final Logger LOG = LoggerFactory.getLogger(TopHashtagBolt.class);
    private static final Integer TOPN = 20;
    private static final Integer TICK_FREQUENCY = 10;

    @Override
    public void execute(Tuple tuple, BasicOutputCollector collector) {
        if (isTickTuple(tuple)) {
            LOG.debug("Tick: " + rankings);
            collector.emit(new Values(new ArrayList(rankings)));
        } else {
            rankHashtag(tuple);
        }
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("tophashtags"));
    }

    @Override
    public Map<String, Object> getComponentConfiguration() {
        Config conf = new Config();
        conf.put(Config.TOPOLOGY_TICK_TUPLE_FREQ_SECS, TICK_FREQUENCY);
        return conf;
    }

    private void rankHashtag(Tuple tuple) {
        String hashtag = tuple.getStringByField("hashtag");
        Integer existingIndex = find(hashtag);
        if (null != existingIndex)
            rankings.set(existingIndex, tuple.getValues());
        else
            rankings.add(tuple.getValues());

        Collections.sort(rankings, new Comparator<List>() {
            @Override
            public int compare(List o1, List o2) {
                return compareRanking(o1, o2);
            }
        });

        shrinkRanking();
    }

    private Integer find(String hashtag) {
        for(int i = 0; i < rankings.size(); ++i) { String current = (String) rankings.get(i).get(0); if (current.equals(hashtag)) { return i; } } return null; } private int compareRanking(List one, List two) { long valueOne = (Long) one.get(1); long valueTwo = (Long) two.get(1); long delta = valueTwo - valueOne; if(delta > 0) {
            return 1;
        } else if (delta < 0) { return -1; } else { return 0; } } private void shrinkRanking() { int size = rankings.size(); if (TOPN >= size) return;
        for (int i = TOPN; i < size; i++) {
            rankings.remove(rankings.size() - 1);
        }
    }

    private static boolean isTickTuple(Tuple tuple) {
        return tuple.getSourceComponent().equals(Constants.SYSTEM_COMPONENT_ID)
                &amp;&amp; tuple.getSourceStreamId().equals(Constants.SYSTEM_TICK_STREAM_ID);
    }
}
{% endcodeblock %}

#### 6b. Save results to Cassandra

Finally, we are storing the top N hashtags per day in Cassandra. For that we&#8217;re using the **row partitioning** pattern to store a row per day and the top hashtags for each time bucket (20 seconds)

{% codeblock lang:java %}
public class TopHashtagToCassandraBolt extends CassandraBaseBolt {
    private static final Logger LOG = LoggerFactory.getLogger(TopHashtagToCassandraBolt.class);

    public TopHashtagToCassandraBolt(Properties properties) {
        super(properties);
    }

    @Override
    public void execute(Tuple tuple, BasicOutputCollector collector) {
        List<List> rankings = (List) tuple.getValue(0);

        Map<String, Long> rankingMap = new HashMap<>();

        for (List list : rankings) {
            rankingMap.put((String) list.get(0), (Long) list.get(1));
        }

        DateFormat df = new SimpleDateFormat("yyyy-MM-dd");

        Statement statement = QueryBuilder.insertInto("top_hashtag_by_day")
                .value("date", df.format(new Date()))
                .value("bucket_time", QueryBuilder.raw("dateof(now())"))
                .value("ranking", rankingMap);

        LOG.debug(statement.toString());

        session.execute(statement);
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {

    }
}
{% endcodeblock %}

### Next steps

Although the hashtag counter may work, I will not say that is entirely correct and there are better ways to do it:

  * You can take a look to this excellent resource: <a href="http://www.michael-noll.com/blog/2013/01/18/implementing-real-time-trending-topics-in-storm/" target="_blank">http://www.michael-noll.com/blog/2013/01/18/implementing-real-time-trending-topics-in-storm/</a>
  * Using **Storm Trident**: In the next post I will show how to use the high level abstraction from Storm that allows to process a stream as a sequence of small batches of data (aka **micro-batching**) and fits better for the top hashtags example.