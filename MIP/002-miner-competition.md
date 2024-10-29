# MIP-2: Subnet Miner Competition

## Preamble

- **MIP Number**: #2
- **Title**: Subnet Miner Competition
- **Author(s)**: @grantdfoster
- **Type**: Standards Track
- **Status**: Draft
- **Created**: 29-10-2024
- **Version**: 0.1

## Summary

We need to have a centralized, documented conversation around improving miner competition on the subnet. This MIP will both capture the current problems and propose solutions.

In the current iteration of the subnet `v1.0.2`, the validators define a "maximum amount of work" a miner can do. This is problematic for miner competition, as the ceiling is quickly reached my the majority of the miners. This issue is compounded by the fact that the validator is selecting miners at random for scoring. The result is that emissions per miner become more "random" than "earned", and competition is stifled.

This MIP will focus on the pieces of logic that contribute to the issue, and the proposed implementation(s) for their resolution.

## Motivation

The subnet metagraph quickly reaches a plateau on each release of the subnet software - meaning that all miners are quickly able to achieve the maximum amount of work, as defined by the validators. The result is that, due to our randomness in how miners are selected for volume scoring, emmissions become more "random" than "earned".

## Specification

1.  Remove pre-defined validator tweet count

    We need to remove the concept of a pre-defined tweet count the validator expects. The validator will now only supply the query, and the miners race to scrape as many tweets as possible based on that query, before the timeout is reached.

2.  Derive a dynamic method for fetching trending queries

    To support this idea of maximum competition, we need to derive a dynamic method for fetching trending queries. The validator needs to supply the miner(s) with queries that have enough volume to drive competition. If a query only returns `10` tweets, that is the maximum amount of work a miner can do for that query.

3.  Store all valid tweet IDs each miner responds with

    We want to ensure miners are not simply responding with stored tweets for a given query and date range. We can do this by storing all valid tweet IDs each miner responds with, and only counting unique tweets by miner.

4.  Reduce randomness in miner selection

    Validators need to iterate through the entire miner list before repeating. This will reduce the randomness in miner selection, and ensure that miners are not unfairly penalized for being skipped. The validator can still "randomly iterate", but must exhaust the list of UIDs before repeating.

5.  Increase timeout from "8" seconds to "20" seconds

    Given the increased volume validators will be asking miners to scrape, we should also increase the response timeout. It currently sits at "8" seconds, but should be increased to a more reasonable timeframe.

## Rationale

It is important to remove this idea of a hard-coded work ceiling. By defining the "maximum amount of work" a miner can do, we are also defining a ceiling that is quickly reached. The issues are two fold - it both kills miner competition and our incentive mechanism at large. Due to the randomness in how miners selected for scoring, emissions per miner become more "random" than "earned". The result is that miners are "randomly" de-registered from the metagraph, discouraging participation and competition.

## Backwards Compatibility

We will need to preserve the `count` and `keywords` property defined in `config/twitter.json` until every validator has updated to the new software release that implements this MIP.

## Test Cases

We can test this implementation on our testnet by running a few validators alongside a handful of miners. The miners can independently compete to scrape as many tweets as possible, and we can analyze the resulting emissions from the metagraph. The scoring logic should remain the same - powered by a kurtosis curve - we are only focused on miner competition.

## Implementation

1.  Remove the concept of a pre-defined tweet count, specifically in [the scoring logic](https://github.com/masa-finance/masa-bittensor/blob/main/masa/validator/forwarder.py#L159). Easiest implementation is to simply set the `count` to a very high number. This allows organic requests (which use the same `Synapse`) to still be driven off of a count.

    ```python
    request = RecentTweetsSynapse(query=query, count=1000000)
    ```

2.  Derive a dynamic method for fetching trending queries. The validator will need to supply the miner(s) with queries that have enough volume to drive competition. The validator currently fetches `keywords` from `config/twitter.json` in the repository. We need to update [this function](https://github.com/masa-finance/masa-bittensor/blob/main/masa/validator/forwarder.py#L131-L147) to fetch a dynamic list of queries.

    ```python
    async def fetch_twitter_config(self):
            async with aiohttp.ClientSession() as session:
                url = self.validator.config.neuron.twitter_config_url
                async with session.get(url) as response:
                    if response.status == 200:
                        configRaw = await response.text()
                        config = json.loads(configRaw)
                        self.validator.keywords = config["keywords"]
                        self.validator.count = int(config["count"])
                    else:
                        self.validator.keywords = ["crypto", "bitcoin", "masa"]
                        self.validator.count = 3
    ```

3.  Store all valid tweet IDs each miner responds with, only counting unique tweets by miner. We must add a piece of state in the validator state to record hotkey and list of tweet ids that hotkey has scraped. This is better than using a `UID` as that is dynamic and can be occupied by many different miners over time. Code must be updated in [this function](https://github.com/masa-finance/masa-bittensor/blob/main/masa/validator/forwarder.py#L149-L297) to support this. Validator state can be updated [here](https://github.com/masa-finance/masa-bittensor/blob/main/masa/base/validator.py#L442-L482)

    ```python
    # Initialize the state
    self.tweets_by_hotkey = {}

    def add_tweet_id(hotkey: str, tweet_id: int):
      if hotkey not in self.tweets_by_hotkey:
        self.tweets_by_hotkey[hotkey] = set()
        self.tweets_by_hotkey[hotkey].add(tweet_id)

    # Example usage
    add_tweet_id('5GhPY8W7PbsXimGf7HQTuXpRsPtrjvVcRkd8j8ddkFQ1wDEz', 17098359098246020)

    # Convert sets to lists for final representation
    tweets_by_hotkeys = {k: list(v) for k, v in self.tweets_by_hotkey.items()}
    print(tweets_by_hotkeys) # Output: {'5GhPY8W7PbsXimGf7HQTuXpRsPtrjvVcRkd8j8ddkFQ1wDEz': [17098359098246020, 17098359098246024]}
    ```

4.  Reduce randomness in miner selection. We need to update [this function](https://github.com/masa-finance/masa-bittensor/blob/main/masa/utils/uids.py#L49) to take an additional argument - the miner uids that have already been called in any given "round". Once there are no more UIDs to call, we reset the "round" for that particular validator. We can specficially compare the available UIDs [here](https://github.com/masa-finance/masa-bittensor/blob/main/masa/utils/uids.py#L80-L82) with a stored list of UIDs. No real need to persist this array in the validator state, it can start fresh on a fresh run.

    ```python
    if not hasattr(self, 'called_uids'):
        self.called_uids = set()

    uids = torch.tensor(version_checked_uids)
    available_uids = [uid for uid in version_checked_uids if uid not in self.called_uids]

    if len(available_uids) == 0:
        self.called_uids = set()
        available_uids = version_checked_uids

    self.called_uids.update(available_uids)

    uids = torch.tensor(available_uids)
    return uids
    ```

5.  Increase timeout from "8" seconds to "20" seconds, or an amount of time that is more reflective of the increased volume of tweets expected to be scraped. Can pass a `timeout` argument on [this line](https://github.com/masa-finance/masa-bittensor/blob/main/masa/validator/forwarder.py#L161-L163)

    ```python
    responses, miner_uids = await self.forward_request(
        request, sample_size=self.validator.config.neuron.sample_size_volume, timeout=20
    )
    ```

## Copyright

This work is licensed under the Apache-2.0 License.
