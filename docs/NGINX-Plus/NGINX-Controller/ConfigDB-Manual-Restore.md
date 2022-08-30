### Set Variables
```bash
CTLRFQDN='nctest.itian.ml'
CTLRAUTHCOOKIE='/var/tmp/cookie.txt'
CTLRAUTHUSER='admin@nginx.com'
CTLRAUTHPASS='1234qwer'
```

### Authentication
```bash
# Client Authentication
curl -sk -c ${CTLRAUTHCOOKIE} -X POST --url "https://${CTLRFQDN}/api/v1/platform/login" \
    --header 'Content-Type: application/json' \
	--data '{"credentials": {"type": "BASIC","username": "'"${CTLRAUTHUSER}"'","password": "'"${CTLRAUTHPASS}"'"}}'

# Controller Authentication Check
curl -sk -b ${CTLRAUTHCOOKIE} -c ${CTLRAUTHCOOKIE} \
-X GET --url "https://${CTLRFQDN}/api/v1/platform/login" | jq
```


### TEST Configs POST API requests from KMA - LG CNS

```bash
curl -sk -c ${CTLRAUTHCOOKIE} -X PUT \
--url "https://${CTLRFQDN}/api/v1/services/environments/hq-env" \
    --header 'Content-Type: application/json' \
	--data '{
  "metadata": {
    "name": "hq-env",
    "displayName": "",
    "description": "",
    "tags": []
  },
  "desiredState": {}
}'
```

```bash
curl -sk -c ${CTLRAUTHCOOKIE} -X PUT \
--url "https://${CTLRFQDN}/api/v1/services/environments/hq-env/gateways/hq-gw" \
--header 'Content-Type: application/json' \
--data '{
  "metadata": {
    "name": "hq-gw",
    "tags": []
  },
  "desiredState": {
    "ingress": {
      "uris": {
        "http://apigw.comis5.kma.go.kr": {}
      },
      "placement": {
        "instanceRefs": [
          {
            "ref": "/infrastructure/locations/unspecified/instances/nginxgw-21-8vxcf"
          },
          {
            "ref": "/infrastructure/locations/unspecified/instances/nginxgw-21-frqh6"
          }
        ]
      }
    }
  }
}'
```



```bash
curl -sk -c ${CTLRAUTHCOOKIE} -X PUT \
--url "https://${CTLRFQDN}/api/v1/services/environments/hq-env/apps/aws-capi" \
--header 'Content-Type: application/json' \
--data '{
  "metadata": {
    "name": "aws-capi",
    "displayName": "",
    "description": "",
    "tags": []
  },
  "desiredState": {}
}'
```

```bash
curl -sk -c ${CTLRAUTHCOOKIE} -X PUT \
--url "https://${CTLRFQDN}/api/v1/services/environments/hq-env/apps/aws-capi/components/api.cgi-bin.aws3.nph-aws3_ana1" \
--header 'Content-Type: application/json' \
--data '{
  "metadata": {
    "name": "api.cgi-bin.aws3.nph-aws3_ana1",
    "tags": []
  },
  "desiredState": {
    "ingress": {
      "gatewayRefs": [
        {
          "ref": "/services/environments/hq-env/gateways/hq-gw"
        }
      ],
      "uris": {
        "/capi/cgi-bin/aws3/nph-aws3_ana1": {
          "matchMethod": "PREFIX"
        }
      }
    },
    "backend": {
      "ntlmAuthentication": "DISABLED",
      "preserveHostHeader": "DISABLED",
      "workloadGroups": {
        "aws-capi-wg": {
          "loadBalancingMethod": {
            "type": "ROUND_ROBIN"
          },
          "uris": {
            "http://aws-capi:8090": {
              "isBackup": false,
              "isDown": false,
              "isDrain": false
            }
          }
        }
      }
    },
    "logging": {
      "errorLog": "DISABLED",
      "accessLog": {
        "state": "DISABLED"
      }
    },
    "publishedApiRefs": [
      {
        "links": {
          "rel": ""
        },
        "ref": "/services/environments/hq-env/apps/aws-capi/published-apis/api.cgi-bin.aws3.nph-aws3_ana1"
      }
    ],
    "security": {
      "identityProviderRefs": []
    }
  }
}'
```


```bash
curl -sk -c ${CTLRAUTHCOOKIE} -X PUT \
--url "https://${CTLRFQDN}/api/v1/security/identity-providers/url_api" \
--header 'Content-Type: application/json' \
--data '{
  "metadata": {
    "name": "url_api",
    "tags": []
  },
  "desiredState": {
    "environmentRefs": [
      {
        "ref": "/services/environments/hq-afso-env"
      }
    ],
    "identityProvider": {
      "type": "API_KEY"
    }
  }
}'
```

```bash
curl -sk -c ${CTLRAUTHCOOKIE} -X PUT \
--url "https://${CTLRFQDN}/api/v1/security/identity-providers/url_api/clients" \
--header 'Content-Type: application/json' \
--data '{
  "items": [
    {
      "metadata": {
        "name": "afsu00000042"
      },
      "desiredState": {
        "credential": {
          "apiKey": "4cd9d2c1f739c5e2b3204fa56579377a95f81ac2aaf18d8bd5f47c8e414fcf8d9642574c2d64dcb3c9511d553e668ad3292a68e3e76e4b597f5bb9a0ee929bf8",
          "type": "API_KEY"
        }
      }
    },
    {
      "metadata": {
        "name": "afsu00000046"
      },
      "desiredState": {
        "credential": {
          "apiKey": "42c832cfd1a4098d1155bbc8262d6841f8085d7dbc0e2c08db9ad35541be96d187b894b0ac32373a484734f530a039552a5055b9730ce27fbd819b5522446de2",
          "type": "API_KEY"
        }
      }
    },
    {
      "metadata": {
        "name": "afsu00000047"
      },
      "desiredState": {
        "credential": {
          "apiKey": "7318c7026f474149282469a395622716e128f0ca5d84e9a689ecd4347d1131cb26c1e41a12d1c153235c61c4715c013972b9d1b5ee1a210ae27c2584ef0342c3",
          "type": "API_KEY"
        }
      }
    },
    {
      "metadata": {
        "name": "test"
      },
      "desiredState": {
        "credential": {
          "apiKey": "9d2c1f739c5e2b3204fa50bbc82626f4741492824684e9a62c08db9ad35541be96d187b894b0ac32373a484734f530a039552a5055b9730ce27fbd819b5522446de2o5o345f3434cr3434gh45gv34554",
          "type": "API_KEY"
        }
      }
    }
  ]
}'
```
