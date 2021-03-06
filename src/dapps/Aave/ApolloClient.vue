<template>
  <div></div>
</template>

<script>
import { WebSocketLink } from 'apollo-link-ws';
import { InMemoryCache } from 'apollo-cache-inmemory';
import gql from 'graphql-tag';
import { SubscriptionClient } from 'subscriptions-transport-ws';
import ApolloClient from 'apollo-client';

export default {
  props: {
    address: {
      type: String,
      default: ''
    }
  },
  mounted() {
    const GRAPHQL_ENDPOINT =
      'wss://api.thegraph.com/subgraphs/name/aave/protocol-raw';

    const client = new SubscriptionClient(GRAPHQL_ENDPOINT, {
      reconnect: true
    });

    const wsLink = new WebSocketLink(client);

    this.apolloClient = new ApolloClient({
      link: wsLink,
      cache: new InMemoryCache()
    });

    this.getReserveData();
    this.getUserData();
    this.getUsdPriceEth();
  },
  methods: {
    getLiquidityRateHistoryUpdate(param) {
      const vm = this;
      this.apolloClient
        .subscribe({
          query: gql`
            subscription LiquidityRateHistoryUpdate($reserveAddress: String!) {
              reserveParamsHistoryItems(
                where: { reserve: $reserveAddress }
                first: 10
              ) {
                variableBorrowRate
                stableBorrowRate
                liquidityRate
                timestamp
              }
            }
            fragment BorrowRateHistoryData on ReserveParamsHistoryItem {
              variableBorrowRate
              stableBorrowRate
              timestamp
              __typename
            }
          `,
          variables: {
            reserveAddress: param
          }
        })
        .subscribe({
          next(resp) {
            vm.$emit(
              'liquidityRateHistory',
              resp.data.reserveParamsHistoryItems
            );
          }
        });
    },
    getUsdPriceEth() {
      const vm = this;
      this.apolloClient
        .subscribe({
          query: gql`
            subscription UsdPriceEth {
              priceOracle(id: "1") {
                usdPriceEth
              }
            }
          `,
          variables: {}
        })
        .subscribe({
          next(resp) {
            vm.$emit('usdPriceEth', resp.data.priceOracle.usdPriceEth);
          }
        });
    },
    getUserData() {
      const vm = this;
      this.apolloClient
        .subscribe({
          query: gql`
            subscription UserPositionUpdateSubscription($userAddress: String!) {
              userReserves(where: { user: $userAddress }) {
                ...UserReserveData
                __typename
              }
            }

            fragment UserReserveData on UserReserve {
              principalATokenBalance
              userBalanceIndex
              redirectedBalance
              interestRedirectionAddress
              reserve {
                id
                name
                symbol
                decimals
                liquidityRate
                lastUpdateTimestamp
                aToken {
                  id
                }
              }
              usageAsCollateralEnabledOnUser
              borrowRate
              borrowRateMode
              originationFee
              principalBorrows
              variableBorrowIndex
              lastUpdateTimestamp
              __typename
            }
          `,
          variables: {
            userAddress: this.address
          }
        })
        .subscribe({
          next(resp) {
            vm.$emit('userReserveData', resp.data.userReserves);
          }
        });
    },
    getReserveData() {
      const vm = this;
      this.apolloClient
        .subscribe({
          query: gql`
            subscription ReserveUpdateSubscription {
              reserves {
                ...ReserveData
                __typename
              }
            }

            fragment ReserveData on Reserve {
              id
              name
              symbol
              decimals
              isActive
              usageAsCollateralEnabled
              borrowingEnabled
              stableBorrowRateEnabled
              baseLTVasCollateral
              liquidityIndex
              reserveLiquidationThreshold
              variableBorrowIndex
              aToken {
                id
                __typename
              }
              availableLiquidity
              stableBorrowRate
              liquidityRate
              totalBorrows
              totalBorrowsStable
              totalBorrowsVariable
              totalLiquidity
              utilizationRate
              variableBorrowRate
              price {
                priceInEth
                __typename
              }
              lastUpdateTimestamp
              __typename
            }
          `,
          variables: {}
        })
        .subscribe({
          next(resp) {
            vm.$emit('reserveData', resp.data.reserves);
          }
        });
    }
  }
};
</script>
