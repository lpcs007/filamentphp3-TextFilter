```php
<?php

namespace App\Components\Filters;

use Filament\Tables\Filters\Indicator;
use Filament\Tables\Filters\BaseFilter;
use Filament\Forms\Components\TextInput;
use Illuminate\Database\Eloquent\Builder;

class TextFilter extends BaseFilter
{
    protected string|array|null $attribute = null;

    protected function setUp(): void
    {
        parent::setUp();

        $this->indicateUsing(function (TextFilter $filter, array $state): array {
            $value = $state['value'] ?? null;

            if (blank($value)) {
                return [];
            }

            $indicator = $filter->getIndicator();
            if (! $indicator instanceof Indicator) {
                $indicator = Indicator::make("{$filter->getLabel()}: {$value}");
            }

            return [$indicator];
        });
    }

    public function getFormField(): TextInput
    {
        return TextInput::make('value')
            ->label($this->getLabel());
    }

    public function attribute(string|array $name): static
    {
        $this->attribute = $name;
        return $this;
    }

    public function getAttribute(): string|array
    {
        return $this->attribute ?? $this->getName();
    }

    public function apply(Builder $query, array $data = []): Builder
    {
        if ($this->hasQueryModificationCallback()) {
            return parent::apply($query, $data);
        }

        $value = $data['value'] ?? null;

        if (blank($value)) {
            return $query;
        }

        $attributes = $this->getAttribute();

        if (is_array($attributes)) {
            $query->where(function (Builder $query) use ($attributes, $value) {
                foreach ($attributes as $attribute) {
                    if (str_contains($attribute, '.')) {
                        [$relation, $relationAttribute] = explode('.', $attribute, 2);
                        $query->orWhereHas($relation, function (Builder $query) use ($relationAttribute, $value) {
                            $query->where($relationAttribute, 'like', '%' . $value . '%');
                        });
                    } else {
                        $query->orWhere($attribute, 'like', '%' . $value . '%');
                    }
                }
            });
        } else {
            if (str_contains($attributes, '.')) {
                [$relation, $relationAttribute] = explode('.', $attributes, 2);
                $query->whereHas($relation, function (Builder $query) use ($relationAttribute, $value) {
                    $query->where($relationAttribute, 'like', '%' . $value . '%');
                });
            } else {
                $query->where($attributes, 'like', '%' . $value . '%');
            }
        }

        return $query;
    }

    public function getActiveCount(): int
    {
        $value = $this->getState()['value'] ?? null;
        return filled($value) ? 1 : 0;
    }
}
```
