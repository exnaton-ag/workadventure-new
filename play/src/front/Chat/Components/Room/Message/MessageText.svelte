<script lang="ts">
    import { Readable, Unsubscriber } from "svelte/store";
    import { Marked } from "marked";
    import { onDestroy, onMount, createEventDispatcher } from "svelte";
    import { ChatMessageContent } from "../../../Connection/ChatConnection";
    import { sanitizeHTML } from "./WA-HTML-Sanitizer";
    export let content: Readable<ChatMessageContent>;
    export let hasDepth: false;

    const dispatch = createEventDispatcher();

    async function getMarked(body: string): Promise<Marked> {
        let marked: Marked;

        // Let's lazy load hljs and markedHighlight if the message contains ```.
        if (body.includes("```")) {
            const hljsPromise = import("highlight.js");
            const markedHighlightPromise = import("marked-highlight");
            const [hljsModule, markedHighlightModule] = await Promise.all([hljsPromise, markedHighlightPromise]);

            marked = new Marked(
                markedHighlightModule.markedHighlight({
                    langPrefix: "hljs language-",
                    highlight(code, lang) {
                        const language = hljsModule.default.getLanguage(lang) ? lang : "plaintext";
                        return hljsModule.default.highlight(code, { language }).value;
                    },
                })
            );
        } else {
            marked = new Marked();
        }

        // Custom renderer for links
        const renderer = new marked.Renderer();
        renderer.link = (href: string, title: string, text: string) => {
            const titleAttr = title ? `title="${title}"` : "";
            return `<a href="${href}" target="_blank" rel="noopener noreferrer" ${titleAttr} style="color: white;">${text}</a>`;
        };

        // Apply the custom renderer and enable line breaks
        marked.use({
            renderer,
            breaks: true,
        });

        return marked;
    }

    let html = "";

    let unsubscriber: Unsubscriber | undefined;
    onMount(() => {
        unsubscriber = content.subscribe((value) => {
            let promiseHtml = getMarked(value.body).then((marked) => marked.parse(value.body));
            promiseHtml
                .then((result) => {
                    html = result;
                })
                .catch((error) => {
                    console.error("Failed to parse markdown content", error);
                    html = $content.body;
                })
                .finally(() => {
                    dispatch("updateMessageBody");
                });
        });
    });

    onDestroy(() => {
        if (unsubscriber) {
            unsubscriber();
        }
    });
</script>

<div
    class="message-bubble tw-m-0 {hasDepth ? 'tw-text-xs tw-leading-4' : 'tw-text-sm'} tw-text-white tw-py-1 tw-px-2"
    style="padding-top: 7px;"
    lang=""
>
    {@html sanitizeHTML(html)}
    <style>
        .message-bubble p:last-of-type {
            margin-bottom: 0;
        }
        .message-bubble a {
            text-decoration: underline;
            opacity: 0.75;
        }
    </style>
</div>

<style>
</style>
