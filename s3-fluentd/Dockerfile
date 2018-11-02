FROM fluent/fluentd:v1.2-onbuild

RUN apk add --update --virtual .build-deps \
        sudo \
 && sudo gem install \
        fluent-plugin-s3 \
 && sudo gem sources --clear-all \
 && apk del .build-deps \
 && rm -rf /var/cache/apk/* \
           /home/fluent/.gem/ruby/2.4.0/cache/*.gem