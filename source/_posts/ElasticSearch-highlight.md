---
title: ElasticSearch highlight
date: 2019-05-18 07:36:51
categories:
  - Tech
tags:
  - laravel
  - elasticSearch
---
Laravel Scout ElasticSearch搜索的基础上，搜索到的关键词加高亮的效果。
参考：
[搜索结果高亮][1]
[Scout][2]


### 自定义搜索引擎
<!--more-->
App\Services\EsEngine.php
```php
<?php

namespace App\Services;

use Laravel\Scout\Builder;
use ScoutEngines\Elasticsearch\ElasticsearchEngine;
use Illuminate\Database\Eloquent\Collection;

class EsEngine extends ElasticsearchEngine
{
    public function search(Builder $builder)
    {
        $result =  $this->performSearch($builder, array_filter([
            'numericFilters' => $this->filters($builder),
            'size' => $builder->limit,
        ]));
        return $result;
    }

    /**
     * Perform the given search on the engine.
     *
     * @param  Builder  $builder
     * @return mixed
     */
    protected function performSearch(Builder $builder, array $options = [])
    {
    //定义查询
        $params = [
            'index' => $this->index,
            'type' => $builder->model->searchableAs(),
            'body' => [
                'query' => [
                    'bool' => [
                        'must' => [
                            [
                                'query_string' => [
                                    'query' => "{$builder->query}",
                                ]
                            ]
                        ]
                    ]
                ],
            ]
        ];
        /**
         * 这里使用了 highlight 的配置
         */
        if ($builder->model->searchSettings
            && isset($builder->model->searchSettings['attributesToHighlight'])
        ) {
            $attributes = $builder->model->searchSettings['attributesToHighlight'];
            foreach ($attributes as $attribute) {
                $params['body']['highlight']['fields'][$attribute] = new \stdClass();
            }
        }

        if (isset($options['from'])) {
            $params['body']['from'] = $options['from'];
        }

        if (isset($options['size'])) {
            $params['body']['size'] = $options['size'];
        }

        if (isset($options['numericFilters']) && count($options['numericFilters'])) {
            $params['body']['query']['bool']['must'] = array_merge($params['body']['query']['bool']['must'],
                $options['numericFilters']);
        }

        return $this->elastic->search($params);
    }

    /**
     * Map the given results to instances of the given model.
     *
     * @param  mixed  $results
     * @param  \Illuminate\Database\Eloquent\Model  $model
     * @return Collection
     */
    public function map(Builder $builder, $results, $model)
    {
        if ($results['hits']['total'] === 0) {
            return Collection::make();
        }

        $keys = collect($results['hits']['hits'])
            ->pluck('_id')->values()->all();

        $models = $model->getScoutModelsByIds(
            $builder, $keys
        )->keyBy(function ($model) {
            return $model->getScoutKey();
        });

        return collect($results['hits']['hits'])->map(function ($hit) use ($model, $models) {
            $one = $models[$hit['_id']];
            /*
             * 这里返回的数据，如果有 highlight，就把对应的  highlight 设置到对象上面，highlights对应model里定义的属性
             */
            if (isset($hit['highlight'])) {
                $one->highlights = $hit['highlight'];
            }

            return $one;
        });
    }

}
```

### 修改Scout的Engine 
config/scout.php
```php
'driver' => env('SCOUT_DRIVER', 'es')
```

app/Providers/AppServiceProvider.php
```php
public function boot()
    {
        //
        resolve(EngineManager::class)->extend('es', function($app) {
        //实例化EsEngine
            return new EsEngine(ElasticBuilder::create()
                ->setHosts(config('scout.elasticsearch.hosts'))
                ->build(),
                config('scout.elasticsearch.index')
            );
        });

    }
```

### 添加属性trait

  app/Traits/EsSearchable.php 
  **traits的拼写，查错查了好久**
```php
namespace App\Traits;
trait EsSearchable
{
  //query
    public $searchSettings = [
        'attributesToHighlight' => [
            '*'
        ]
    ];

    //store highlight results for blade template
    public $highlight = [];
}
```
### model中添加 EsSearchabel Trait
App/Models/Topic.php
```php
...
use App\Traits\EsSearchabel;
use EsSearchabel;
...
```

### 添加关键词的样式
进入**Tinker**
```php
//结果中带有<em></em>标签，对标签设置color属性。
$collection = App\Models\Topic::search("qu")->get();
$collection->first()['highlights']['body][0];
```




  [1]: https://learnku.com/articles/4038/tutorial-two-write-a-search-solve-the-search-results-highlight-the-problem-using-laravel-scout-elasticsearch-ik-word-segmentation
  [2]: https://learnku.com/docs/laravel/5.8/scout/3946