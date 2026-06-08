<script setup>
useHead({
    meta: [{ name: 'viewport', content: 'width=device-width, initial-scale=1' }],
    link: [{ rel: 'icon', href: '/favicon.ico' }],
    htmlAttrs: {
        lang: 'en',
    },
});

const title = 'Audio Latency Tester';
const description = 'Test the latency of your audio output devices';

useSeoMeta({
    title,
    description,
    ogTitle: title,
    ogDescription: description,
});
</script>

<template>
    <UApp>
        <div class="flex min-h-dvh flex-col">
            <UHeader>
                <template #left>
                    <h1 class="text-2xl font-bold">{{ title }}</h1>
                </template>

                <template #right>
                    <UColorModeButton />

                    <UButton
                        to="https://github.com/ilkerdurmaz/sound-latency-tester"
                        target="_blank"
                        icon="i-simple-icons-github"
                        aria-label="GitHub"
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
                    <p class="text-sm text-muted">Built by İlker Durmaz</p>
                </template>
            </UFooter>
        </div>
    </UApp>
</template>
