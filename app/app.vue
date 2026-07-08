<script setup>
const { locale, locales, t } = useI18n();
const switchLocalePath = useSwitchLocalePath();
const localeHead = useLocaleHead();

const title = computed(() => t('app.title'));
const description = computed(() => t('app.description'));

useHead(() => ({
    meta: [{ name: 'viewport', content: 'width=device-width, initial-scale=1' }, ...(localeHead.value.meta ?? [])],
    link: [{ rel: 'icon', href: '/favicon.ico' }, ...(localeHead.value.link ?? [])],
    htmlAttrs: {
        lang: localeHead.value.htmlAttrs?.lang,
        dir: localeHead.value.htmlAttrs?.dir,
    },
}));

useSeoMeta(() => ({
    title: title.value,
    description: description.value,
    ogTitle: title.value,
    ogDescription: description.value,
}));
</script>

<template>
    <UApp>
        <div class="flex min-h-dvh flex-col">
            <UHeader>
                <template #left>
                    <h1 class="text-2xl font-bold">{{ title }}</h1>
                </template>

                <template #right>
                    <div class="flex items-center gap-1" :aria-label="t('app.language')">
                        <UButton
                            v-for="availableLocale in locales"
                            :key="availableLocale.code"
                            :to="switchLocalePath(availableLocale.code)"
                            :label="availableLocale.name"
                            size="xs"
                            color="neutral"
                            :variant="availableLocale.code === locale ? 'solid' : 'ghost'"
                            :aria-label="t('app.switchLanguage', { name: availableLocale.name })" />
                    </div>

                    <UColorModeButton />

                    <UButton
                        to="https://github.com/ilkerdurmaz/sound-latency-tester"
                        target="_blank"
                        icon="i-simple-icons-github"
                        :aria-label="t('app.github')"
                        color="neutral"
                        variant="ghost" />
                </template>
            </UHeader>

            <UMain class="flex min-h-0 flex-1 flex-col">
                <NuxtPage />
            </UMain>

            <USeparator />

            <UFooter>
                <template #left>
                    <p class="text-sm text-muted">{{ t('app.builtBy') }}</p>
                </template>
            </UFooter>
        </div>
    </UApp>
</template>
